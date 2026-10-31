# pterodactyl-permission-php8.3# Pterodactyl Permissions Troubleshooting ~ (PHP 8.3)

## Error

> The panel cannot write to /var/www/pterodactyl. Give the www-data user write access to it, then try again.

This usually means the account running the panel's PHP-FPM workers cannot write to the Pterodactyl directory. This guide assumes a standard Linux installation at `/var/www/pterodactyl`, with PHP 8.3-FPM running as `www-data`.

## 1. Verify the PHP-FPM worker account

```bash
sudo grep -E '^\s*(user|group)\s*=' /etc/php/8.3/fpm/pool.d/www.conf
```

If the worker user is not `www-data`, adapt the commands below to your actual PHP-FPM user and group.

## 2. Check current permissions

```bash
ls -ld /var/www/pterodactyl
namei -l /var/www/pterodactyl
sudo -u www-data test -w /var/www/pterodactyl && echo 'Writable' || echo 'Not writable'
```

## 3. Fix ownership and permissions

**Note:** These commands grant the PHP-FPM account ownership of the panel files. That is sometimes required by extension installers, but it increases the impact of a compromised web application. Prefer the extension vendor's narrower permissions guidance when available, and back up the panel first.

```bash
sudo chown -R www-data:www-data /var/www/pterodactyl
sudo find /var/www/pterodactyl -type d -exec chmod 755 {} +
sudo find /var/www/pterodactyl -type f -exec chmod 644 {} +
sudo chmod -R ug+rwX /var/www/pterodactyl/storage /var/www/pterodactyl/bootstrap/cache
```

Do **not** use `chmod -R 777`.

## 4. Clear the panel cache

```bash
cd /var/www/pterodactyl
sudo -u www-data php artisan optimize:clear
```

If your CLI defaults to a different PHP version, use `php8.3` instead of `php`.

## 5. Restart PHP 8.3-FPM and the web server

For Nginx installations:

```bash
sudo systemctl restart php8.3-fpm
sudo systemctl restart nginx
```

If you use Apache, restart Apache instead of Nginx. Restart `pteroq` only if you have also changed queue-worker configuration.

## 6. Verify and retry

```bash
sudo -u www-data test -w /var/www/pterodactyl && echo 'Panel directory writable' || echo 'Permission issue remains'
```

Then retry installing your extension in the panel.

## Troubleshooting

If the error persists:

```bash
ls -ld /var/www /var/www/pterodactyl
namei -l /var/www/pterodactyl
sudo systemctl status php8.3-fpm --no-pager
sudo journalctl -u php8.3-fpm -n 50 --no-pager
```

Confirm that your website is actually using PHP 8.3-FPM, not another PHP-FPM pool or container. Check for filesystem read-only mounts, ACLs, and security policies if ordinary Unix permissions appear correct.

## Security notes

- Back up your panel files and database before installing third-party extensions.
- Only install extensions from trusted sources.
- Avoid world-writable permissions (`777`).
- Once installation is complete, consider restoring tighter permissions for application code while leaving Laravel's `storage` and `bootstrap/cache` writable, provided the extension supports this.
