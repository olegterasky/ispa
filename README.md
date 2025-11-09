# Single-Server cPanel + Mail on AWS (EU-North-1 + SES Frankfurt) — **Runbook**

> Goal: Deploy a single EC2 server running cPanel/WHM + mail with top-tier deliverability, maximum cost-efficiency, and solid security guardrails.
> Regions: **Compute in `eu-north-1` (Stockholm)**, **SES in `eu-central-1` (Frankfurt)**.
> Edge: **Shield Standard** only (automatic). **No CloudFront.**

---

## 0) Quick Facts & Defaults

* **Instance**: `t3.medium` (2 vCPU, 4 GB)
* **AMI**: AlmaLinux 9 x86_64
* **Disk**: 60–80 GB gp3
* **Admin**: **SSM Session Manager** only (no SSH)
* **DNS**: Route 53 public hosted zone
* **Mail**: Exim/Dovecot via cPanel; **outbound relay through Amazon SES (Frankfurt)**
* **Auth**: SPF, DKIM, DMARC, and **PTR** (reverse DNS)
* **Ports**: `80,443,465,587,993,995` (+ optional 25), **WHM 2087** & **cPanel 2083** locked to your IP

Replace placeholders:

```
DOMAIN       = example.com            (or example.co.il)
MAIL_HOST    = mail.example.com       (or mail.example.co.il)
BOUNCE_HOST  = bounce.example.com
EIP          = x.x.x.x
SES_REGION   = eu-central-1
SES_SMTP     = email-smtp.eu-central-1.amazonaws.com
YOUR_ADMIN_IP= A.B.C.D/32
```

---

## 1) Account & Guardrails (Single Account)

1. Enable **MFA** on root; don’t use root for ops.
2. Create IAM role **ec2-cpanel-role** with:

   * `AmazonSSMManagedInstanceCore`
   * `CloudWatchAgentServerPolicy`
3. (Optional) Create **backup role** with write-only access to an S3 backup bucket.
4. Turn on **CloudTrail**, **Config**, **GuardDuty**, **Security Hub**.

---

## 2) Networking & Security Groups

Create/choose a VPC with a **public subnet** and an **Internet Gateway**.

**Security Group: `sg-cpanel`**

```
INBOUND:
  80/tcp    0.0.0.0/0
  443/tcp   0.0.0.0/0
  465/tcp   0.0.0.0/0   # SMTPS
  587/tcp   0.0.0.0/0   # Submission
  993/tcp   0.0.0.0/0   # IMAPS
  995/tcp   0.0.0.0/0   # POP3S
  25/tcp    0.0.0.0/0   # Optional (prefer SES relay)
  2083/tcp  YOUR_ADMIN_IP
  2087/tcp  YOUR_ADMIN_IP
OUTBOUND:
  all       0.0.0.0/0
```

> Keep **22/tcp closed**; use **SSM**.

Allocate an **Elastic IP** and attach later.

---

## 3) Launch EC2 (EU-North-1)

* AMI: **AlmaLinux 9 (x86_64)**
* Type: **t3.medium**
* Disk: **gp3 60–80 GB**
* SG: `sg-cpanel`
* IAM Instance Profile: **ec2-cpanel-role**
* Associate **Elastic IP** after launch

**User data (baseline hardening):**

```bash
#!/bin/bash
dnf -y update
dnf -y install fail2ban
systemctl enable --now fail2ban
# SSM agent is preinstalled on AL9 in AWS images
```

---

## 4) Install cPanel/WHM

Connect via **Session Manager** → run:

```bash
hostnamectl set-hostname MAIL_HOST
setenforce 0
sed -ri 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config

cd /home
curl -o latest -L https://securedownloads.cpanel.net/latest
sh latest
```

Initial WHM wizard:

* Set contact email, nameservers (Route 53), and services (Exim/Dovecot).

---

## 5) Domain & DNS (Route 53)

### 5.1 Hosted Zone

Create a **public hosted zone** for `DOMAIN`.

If your TLD is **`.co.il`** and registered elsewhere, change **nameservers** at the registrar to the 4 NS values from Route 53. Wait for propagation.

### 5.2 Records (minimal set)

**A (mail host)**

```
Name: MAIL_HOST
Type: A
Value: EIP
TTL: 300
```

**MX (domain root)**

```
Name: @
Type: MX
Value: 10 MAIL_HOST.
TTL: 300
```

**SPF (TXT on root)**

```
Name: @
Type: TXT
Value: "v=spf1 ip4:EIP include:amazonses.com -all"
TTL: 300
```

**DMARC (TXT on _dmarc)**

```
Name: _dmarc
Type: TXT
Value: "v=DMARC1; p=quarantine; rua=mailto:dmarc@DOMAIN; ruf=mailto:dmarc@DOMAIN; adkim=s; aspf=s"
TTL: 300
```

> You will add **DKIM** and **MAIL-FROM** DNS records in §6 after configuring SES.

---

## 6) Amazon SES (Frankfurt)

SES is **not** in Stockholm; use **`eu-central-1`**.

1. **Verify domain**: SES → *Verified identities* → **Create identity (Domain)** → `DOMAIN`, **Easy DKIM** (RSA-2048).

   * SES shows **3 DKIM CNAMEs**. Add them to Route 53 (TTL 300).

2. **Custom MAIL-FROM**: In SES identity → **Set MAIL FROM**: `BOUNCE_HOST`.

   * SES shows **MX + TXT** for MAIL-FROM. Add both records.

3. **Request production access**: SES → *Account details* → request out of sandbox.

4. **Create SMTP credentials** (not IAM access keys):

   * Save **SMTP username/password** and region endpoint **`SES_SMTP`**.

**Example DKIM/MAIL-FROM records (you will copy actual values from SES):**

```txt
# DKIM (3 records)
Name: xxxxx._domainkey.DOMAIN.     Type: CNAME   Value: xxxxx.dkim.amazonses.com.
Name: yyyyy._domainkey.DOMAIN.     Type: CNAME   Value: yyyyy.dkim.amazonses.com.
Name: zzzzz._domainkey.DOMAIN.     Type: CNAME   Value: zzzzz.dkim.amazonses.com.

# MAIL-FROM
Name: BOUNCE_HOST.                  Type: MX      Value: 10 feedback-smtp.eu-central-1.amazonses.com.
Name: BOUNCE_HOST.                  Type: TXT     Value: "v=spf1 include:amazonses.com -all"
```

---

## 7) Reverse DNS (PTR)

Open **AWS Support case** → *EC2 – Elastic IP Reverse DNS*.
Provide:

* **Elastic IP**: `EIP`
* **Reverse DNS domain**: `MAIL_HOST`

Ensure forward A record and PTR match (`MAIL_HOST` ↔ `EIP`).

---

## 8) TLS (AutoSSL)

WHM → **Manage AutoSSL** → Provider **Let’s Encrypt** → **Run AutoSSL**.
Confirm valid certs for:

* `DOMAIN`, `MAIL_HOST`, and user vhosts as needed.

---

## 9) Exim: Relay Outbound via SES

WHM → **Exim Configuration Manager → Advanced Editor**.

**AUTH** (add or merge):

```nginx
ses_login:
  driver = plaintext
  public_name = LOGIN
  client_send = : SMTP_USERNAME : SMTP_PASSWORD
```

**PREROUTERS** (place *before* default routers):

```nginx
send_via_ses:
  driver = manualroute
  domains = ! +local_domains
  transport = ses_smtp
  route_list = * SES_SMTP::587
  no_more
```

**TRANSPORT**:

```nginx
ses_smtp:
  driver = smtp
  port = 587
  hosts_require_auth = *
  hosts_require_tls = *
```

Save → WHM restarts Exim.

> Keep port **587** (TLS). No provider ticket to unblock 25 needed.

---

## 10) Hardening

* **cPHulk**: enable (login brute-force protection).
* **CSF** firewall: install/enable; allow only required ports.
* **ModSecurity**: enable + **OWASP CRS**.
* **fail2ban**: jails for exim-auth, dovecot, apache.
* Disable **anonymous FTP**, WebDAV, unneeded services.
* Strong TLS only (TLS 1.2/1.3).
* Per-domain and per-hour **outbound limits** in WHM (Exim configuration).
* **No SSH**; use **SSM**.

---

## 11) Monitoring & Alerts

* **CloudWatch Alarms**:

  * `StatusCheckFailed >= 1`
  * `CPUUtilization > 80%`
  * `EBS BurstBalance < 20%`
* **SNS topic** → your email/Telegram/Webhook.
* **GuardDuty**: enabled for SMTP anomalies / reconnaissance.
* **Log sources**: Exim main/reject logs, cPHulk, CSF, fail2ban, `/var/log/messages`.

---

## 12) Backups & DR

* **EBS snapshots** (AWS Backup) — daily, retain 30–90 days.
* **cPanel account backups** → S3 (SSE-S3/SSE-KMS) → lifecycle to IA/Glacier.
* **Config backup**: `/etc/exim/`, `/etc/valiases/`, `/etc/virtual/`, `/var/named/`.

**Restore drill (monthly):**

1. Launch fresh `t3.medium` in same AZ.
2. Attach/restore EBS from snapshot.
3. Reinstall cPanel license (if needed).
4. Restore cPanel accounts from S3.
5. Swap Elastic IP if you’re replacing the host.

---

## 13) Validation & Deliverability

**DNS checks**

```bash
dig +short A MAIL_HOST
dig +short MX DOMAIN
dig +short TXT DOMAIN          # SPF present?
dig +short TXT _dmarc.DOMAIN   # DMARC present?
# DKIM CNAMEs:
dig +short CNAME xxxxx._domainkey.DOMAIN
```

**Mail flow**

* Send test to **Gmail**/**Outlook**. In Gmail → “Show original”:

  * `SPF=pass` (via amazonses.com)
  * `DKIM=pass` (d=DOMAIN)
  * `DMARC=pass`
* Check **SES → Sending statistics** for deliveries/bounces.
* Add `DOMAIN` to **Gmail Postmaster**.

---

## 14) Maintenance SOP

| Task                          | Cadence   |
| ----------------------------- | --------- |
| OS & cPanel updates           | Weekly    |
| Review SES bounces/complaints | Weekly    |
| Snapshot restore test         | Monthly   |
| Re-check SPF/DKIM/DMARC       | Quarterly |
| Domain renewal                | Yearly    |

---

## 15) Troubleshooting Cheatsheet

* **TLS errors on SMTP** → confirm port 587 open; Exim `hosts_require_tls = *`; server time correct.
* **SPF fails** → TXT syntax; ensure `ip4:EIP` and `include:amazonses.com`.
* **DKIM fails** → CNAME names/values must match SES exactly; wait for TTL.
* **DMARC quarantine** too aggressive → start with `p=none` while testing, then move to `quarantine`, later `reject`.
* **PTR missing** → AWS Support reverse-DNS case not yet applied; confirm A record exists for `MAIL_HOST`.
* **WHM/cPanel ports blocked** → verify SG and CSF allow 2083/2087 from YOUR_ADMIN_IP.

---

## 16) Cut-and-Paste DNS Examples (Route 53)

> Replace placeholders and paste into Route 53 UI. One record per entry.

```
A     MAIL_HOST.        300   EIP
MX    DOMAIN.           300   10 MAIL_HOST.
TXT   DOMAIN.           300   "v=spf1 ip4:EIP include:amazonses.com -all"
TXT   _dmarc.DOMAIN.    300   "v=DMARC1; p=quarantine; rua=mailto:dmarc@DOMAIN; ruf=mailto:dmarc@DOMAIN; adkim=s; aspf=s"

# From SES (examples — use exact values SES shows you)
CNAME xxxxx._domainkey.DOMAIN. 300 xxxxx.dkim.amazonses.com.
CNAME yyyyy._domainkey.DOMAIN. 300 yyyyy.dkim.amazonses.com.
CNAME zzzzz._domainkey.DOMAIN. 300 zzzzz.dkim.amazonses.com.

# MAIL-FROM
MX    BOUNCE_HOST.      300   10 feedback-smtp.eu-central-1.amazonses.com.
TXT   BOUNCE_HOST.      300   "v=spf1 include:amazonses.com -all"
```

---

## 17) Final Checklist (tick before go-live)

* [ ] EC2 `t3.medium` live in **eu-north-1** with EIP
* [ ] Shield Standard (auto) – nothing to do
* [ ] Route 53 hosted zone delegated (NS live)
* [ ] A/MX/SPF/DMARC records published
* [ ] SES domain + DKIM + MAIL-FROM verified in **eu-central-1**
* [ ] SMTP relay configured in Exim (port 587 TLS)
* [ ] PTR set: `MAIL_HOST` ↔ `EIP`
* [ ] AutoSSL valid for `DOMAIN` & `MAIL_HOST`
* [ ] CSF + ModSecurity + cPHulk + fail2ban enabled
* [ ] CloudWatch alarms + SNS wired
* [ ] Backups to S3 + EBS snapshots configured
* [ ] Test emails: SPF/DKIM/DMARC **PASS**

---

If you want, I can also supply a **Terraform starter pack** that creates the EC2, SG, IAM role, EIP, and Route 53 scaffolding matching this runbook.
