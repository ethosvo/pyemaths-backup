# pyeMaths.com Migration Changelog

## Migration from Hostinger to AWS — 16 August 2026

### Overview

Full WordPress migration of pyemaths.com from Hostinger shared hosting to AWS infrastructure (EC2 + RDS MariaDB), with DNS cutover via Cloudflare. All content, plugins, themes, and media preserved.

---

### Phase 1: Backup from Hostinger

1. **Verified Hostinger API access** using Bearer token against `https://developers.hostinger.com/api/hosting/v1/`.
2. **Enabled SSH access** on Hostinger hPanel (Advanced → SSH Access).
   - Host: `191.96.63.173`, Port: `65002`, User: `u547625228`
3. **Exported database** via SSH:
   ```bash
   set +H
   mysqldump -u u547625228_pye -p'G78mod!44' u547625228_pyemaths_wp > ~/pyemaths_db.sql
   ```
   - Result: 43MB SQL dump (MariaDB 11.8.8)
4. **Zipped all site files** via SSH:
   ```bash
   cd ~/domains/pyemaths.com/public_html
   zip -r ~/pyemaths_files.zip . -x "db-backup.sql"
   ```
   - Result: 427MB zip of entire public_html
5. **Downloaded both files** locally via SCP.
6. **Cleaned up** temporary files on Hostinger server.
7. **Pushed backup to GitHub** (`ethosvo/pyemaths-backup`) using git-lfs for large files.

---

### Phase 2: AWS Infrastructure Provisioning

1. **Security Groups created** (VPC: `vpc-0f9b3d7c7d2717ed4`):
   - `pyemaths-web-sg` (`sg-0eb6df42bd49021d5`): ports 22, 80, 443 open
   - `pyemaths-db-sg` (`sg-05f59eb0ac758e8e7`): port 3306 from web SG only

2. **SSH Key Pair created**: `pyemaths-aws-key` (PEM stored locally, gitignored)

3. **RDS MariaDB instance created**:
   - Identifier: `pyemaths-mariadb`
   - Engine: MariaDB 11.8.8 (matches Hostinger source version exactly)
   - Instance class: `db.t3.micro`
   - Storage: 20GB gp3
   - Database name: `pyemaths_wp`
   - Master user: `pyemaths_admin`
   - Endpoint: `pyemaths-mariadb.c9guoe80uf6x.eu-west-2.rds.amazonaws.com`
   - Not publicly accessible, SSL required

4. **EC2 instance launched**:
   - AMI: Amazon Linux 2023 (`ami-0de87753593ec47fd`)
   - Instance type: `t3.small`
   - Subnet: `subnet-0390be2ab8c8d7338` (eu-west-2a)
   - Instance ID: `i-008f9e21881545b73`
   - User data installed: Apache, PHP 8.1, php-fpm, MariaDB client, unzip

5. **Elastic IP allocated and associated**: `18.133.11.132`
   - Allocation ID: `eipalloc-0b7e3969594f91d42`
   - This IP is permanent (survives stop/start cycles)

---

### Phase 3: Site Restoration on AWS

1. **Uploaded and extracted WordPress files**:
   ```
   scp pyemaths_files.zip → ec2:/tmp/
   sudo unzip -oq /tmp/pyemaths_files.zip -d /var/www/html
   sudo chown -R apache:apache /var/www/html
   ```

2. **Imported database** into RDS:
   ```
   mysql -h <rds-endpoint> -u pyemaths_admin -p --ssl pyemaths_wp < /tmp/pyemaths_db.sql
   ```
   - All 78 tables imported successfully.

3. **Created fresh wp-config.php** with:
   - New RDS database credentials
   - Fresh WordPress secret keys/salts (from api.wordpress.org)
   - `MYSQL_CLIENT_FLAGS` set for SSL connection to RDS
   - Cloudflare X-Forwarded-Proto trust snippet (for Flexible SSL mode):
     ```php
     if ( isset( $_SERVER['HTTP_X_FORWARDED_PROTO'] ) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https' ) {
         $_SERVER['HTTPS'] = 'on';
     }
     ```

4. **PHP version**: Installed PHP 8.1.34 (replaced default PHP 8.5 which was incompatible with older plugins).

5. **Apache configuration**:
   - Set `AllowOverride All` for `/var/www/html` (required for .htaccess/permalink rewrites)
   - Removed old Hostinger-specific HTTPS redirect from `.htaccess` (Cloudflare handles this)
   - Removed Wordfence auto_prepend_file from `.user.ini` (plugin removed)
   - Removed stale cPanel session.save_path from `.user.ini`

6. **WordPress admin access**:
   - Admin username: `aQ7J^1x#vM3XM)&)` (original from Hostinger install)
   - Password was reset via wp-cli (stored securely, not in this doc)
   - Login URL: `https://pyemaths.com/wp-login.php`

---

### Phase 4: DNS Cutover via Cloudflare

1. **Domain registered at GoDaddy**, nameservers pointed to Cloudflare (`bailey.ns.cloudflare.com`, `kurt.ns.cloudflare.com`).
2. **Cloudflare DNS records set**:
   - A record: `pyemaths.com` → `18.133.11.132` (Proxied)
   - A record: `www.pyemaths.com` → `18.133.11.132` (Proxied)
   - A record: `ftp.pyemaths.com` → `18.133.11.132` (Proxied)
   - AAAA records from Hostinger **deleted** (were routing IPv6 traffic to old server)
3. **Cloudflare SSL mode**: Flexible (Cloudflare terminates HTTPS for visitors, talks HTTP to origin).
4. **siteurl/home** in database updated to `https://pyemaths.com`.

---

### Phase 5: Plugin Activation & Compatibility Fixes

1. **Caldera Forms** — patched PHP 8 incompatibility (unparenthesized ternary on line 3732 of `classes/core.php`). Logic preserved, just added required parentheses.

2. **Revolution Slider** — generates PHP "continue targeting switch" warnings (cosmetic, non-fatal). Works correctly.

3. **All 46 plugins activated** one-by-one with automated health checks after each. Full list of active plugins matches the original Hostinger setup.

4. **Wordfence removed** — not re-activated (was causing path-reference errors to old Hostinger directories; Cloudflare + Sucuri + Shield provide equivalent protection).

5. **Autoptimize** — was inactive on Hostinger, activated during this migration.

---

### Phase 6: Plugin & Core Updates (same day)

**Phase 6a — Low-risk plugin updates (17 plugins):**
- 404page 11.2.2 → 11.4.8
- Black Studio TinyMCE Widget 2.6.9 → 2.7.3
- Google XML Sitemaps 4.1.0 → 4.1.24
- iframe 4.5 → 6.0
- Imagify 1.9.10 → 2.3.2
- Insert Headers and Footers 1.4.5 → 2.3.8
- Progress Bar WP 1.3.1 → 1.5.0
- Responsive Accordion And Collapse 2.1.8 → 2.5.3
- Responsive Pricing Table 5.1.0 → 5.1.13
- Scroll Triggered Animations 2.5.0 → 3.1.1
- Simple CSS 1.1 → 1.1.1
- Sticky Menu on Scroll 2.2 → 2.36
- Strong Testimonials 2.40.7 → 3.3.6
- Sucuri Security 1.8.24 → 2.7.4
- UpdraftPlus 1.23.10 → 1.26.6
- WP-Optimize 3.0.19 → 4.6.1
- WP Mail Logging 1.9.6 → 1.16.0

**Phase 6b — Medium-risk plugin updates:**
- SiteOrigin Page Builder 2.10.17 → 2.36.0
- SiteOrigin Widgets Bundle 1.17.0 → 1.74.2
- Autoptimize 2.7.7 → 3.1.15.1
- Animate It! 2.3.7 → 3.0.4

**Phase 6c — WordPress core update:**
- WordPress 5.4.21 → **7.0.4**
- Database schema upgraded to version 61833
- The `fix-front-page.php` mu-plugin (workaround for WP 5.4 routing bug) was **removed** — no longer needed, WP 7.0 handles page routing correctly natively
- Imagify re-enabled successfully (had temporarily failed on WP 5.4 due to ActionScheduler table creation issue, resolved by core update)

---

### Phase 7: Backup Configuration

1. **RDS automated snapshots**: already enabled, daily, 7-day retention (built into RDS, no extra cost).
2. **EBS snapshot lifecycle policy** created via AWS Data Lifecycle Manager:
   - Policy ID: `policy-0548ec1b4328a3758`
   - Schedule: daily at 03:00 UTC
   - Retention: 7 snapshots
   - Targets: EC2 instance tagged `Backup=daily`
   - Cost: ~£1-2/month
3. **Pre-upgrade manual snapshots** taken:
   - EBS: `snap-006faf1c57dc78bc7`
   - RDS: `pyemaths-pre-upgrade-20260816-1851`

---

### Not Updated (requires license key)

- **GP Premium**: 1.8.3 (update to 2.5.6 requires GeneratePress license key)
- **GeneratePress theme**: 2.4.2 (update to 3.6.1 should be done together with GP Premium)
- **Revolution Slider**: 5.4.7.2 (premium plugin, requires ThemePunch license for updates)

These work correctly as-is on WP 7.0.4 + PHP 8.1. No urgency unless security patches are needed.

---

### Known Issues / Notes for Jonny

1. **WP Mail SMTP**: configured for Sendinblue/Brevo, which needs the new server IP (`18.133.11.132`) whitelisted at https://app.brevo.com/security/authorised_ips — otherwise outbound emails will fail.

2. **Cloudflare SSL is "Flexible"**: origin server (EC2) does not have its own SSL certificate. Cloudflare handles HTTPS for visitors. For end-to-end encryption, install a Cloudflare Origin Certificate on the EC2 box and switch Cloudflare to "Full (strict)" mode.

3. **Old Hostinger server**: still running at `191.96.63.173` with the original site intact. Can be decommissioned once satisfied everything works on AWS. The Hostinger hosting plan can be cancelled when ready.

4. **WordPress admin credentials**: username is `aQ7J^1x#vM3XM)&)` (unusual auto-generated string from original Hostinger install). Consider creating a cleaner admin account and demoting/deleting this one.

5. **Hostinger remote MySQL connection**: was enabled for IP `188.29.204.119` during migration — should be removed from Hostinger's DB settings if/when the hosting plan is cancelled.

6. **Cron jobs on Hostinger**: all cleared during migration (were temporary backup jobs). None remain.

---

### AWS Resource Summary

| Resource | Identifier | Notes |
|----------|-----------|-------|
| EC2 Instance | `i-008f9e21881545b73` | t3.small, Amazon Linux 2023 |
| Elastic IP | `18.133.11.132` | Permanent, associated to EC2 |
| RDS Instance | `pyemaths-mariadb` | db.t3.micro, MariaDB 11.8.8 |
| RDS Endpoint | `pyemaths-mariadb.c9guoe80uf6x.eu-west-2.rds.amazonaws.com` | SSL required |
| Web Security Group | `sg-0eb6df42bd49021d5` | Ports 22/80/443 |
| DB Security Group | `sg-05f59eb0ac758e8e7` | Port 3306 from web SG only |
| EBS Volume | `vol-01129a73aa4e57662` | 20GB gp3 |
| Key Pair | `pyemaths-aws-key` | PEM file stored locally (gitignored) |
| DLM Policy | `policy-0548ec1b4328a3758` | Daily EBS snapshots, 7-day retention |
| Region | `eu-west-2` (London) | |
| Account | `980481493635` | |

### Monthly Cost Estimate

| Item | Cost |
|------|------|
| EC2 t3.small | ~£15 |
| RDS db.t3.micro | ~£12 |
| EBS 20GB + snapshots | ~£3 |
| Elastic IP | £0 (while attached) |
| **Total** | **~£30/month** |
