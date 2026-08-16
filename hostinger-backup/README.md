# hostinger-backup

Full backup of the pyemaths.com WordPress site pulled directly from Hostinger via SSH, ready for restore on AWS infrastructure.

## Contents

- `pyemaths_db.sql` — full MySQL/MariaDB dump of `u547625228_pyemaths_wp` (43MB)
- `pyemaths_files.zip` — full `public_html` directory (WordPress core, themes, plugins, uploads) (427MB)
- `wp-config.php.reference` — **not committed to git** (contains DB credentials and secret keys). Kept locally only, for reference when configuring the new environment.

## How this backup was taken

1. Enabled SSH access in Hostinger hPanel (Advanced → SSH Access).
2. Connected via `ssh -p 65002 u547625228@<host-ip>`.
3. On the server:
   ```bash
   set +H   # disable bash history expansion (DB password contains '!')
   mysqldump -u u547625228_pye -pG78mod!44 u547625228_pyemaths_wp > ~/pyemaths_db.sql
   cd ~/domains/pyemaths.com/public_html
   zip -r ~/pyemaths_files.zip . -x "db-backup.sql"
   ```
4. Pulled both files down locally with `scp`.
5. Deleted the temp dump/zip files from the Hostinger server afterward.

## Restoring on AWS

General approach (e.g. EC2 + RDS, or Lightsail):

1. Provision a LAMP/LEMP stack (PHP 8.x, MySQL/MariaDB, Apache or Nginx).
2. Create a new database and user, import the dump:
   ```bash
   mysql -u <user> -p <new_db_name> < pyemaths_db.sql
   ```
3. Unzip `pyemaths_files.zip` into the new web root (e.g. `/var/www/html`).
4. Create a new `wp-config.php` on the target server with:
   - `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_HOST` pointing at the new database
   - Fresh secret keys/salts (generate at https://api.wordpress.org/secret-key/1.1/salt/)
   - Do **not** reuse the old wp-config.php verbatim without rotating DB credentials and salts
5. Update the site URL in the database if the domain or environment changes:
   ```sql
   UPDATE wp_options SET option_value = 'https://newdomain.com' WHERE option_name IN ('siteurl','home');
   ```
6. Set correct file ownership/permissions for the web server user.
7. Point DNS at the new AWS environment once verified.

## Security note

The old Hostinger DB password (`G78mod!44`) and any credentials found in `wp-config.php.reference` should be treated as compromised for reuse — generate new credentials for the AWS environment rather than copying these over.
