# Visitor analytics — self-hosted GoatCounter

`fuzue.tech` is a static page on GitHub Pages, so there are no server logs to read.
Analytics therefore come from a small script in `index.html` that reports to our own
GoatCounter instance at **https://stats.fuzue.tech**, running on `vaz.io`.

Why GoatCounter: one static Go binary against SQLite, EUPL-1.2, no cookies and no
personal data stored — so no consent banner is needed — and the counting script is
~9 KB, which keeps us inside the site's performance budget.

## Deployed layout

| Piece | Where |
|---|---|
| Counting snippet | `index.html`, just before `</body>` |
| Binary | `/usr/local/bin/goatcounter` (v2.7.0) |
| Service | `goatcounter.service`, listening on `127.0.0.1:8099` |
| TLS + proxy | Apache vhost `stats.fuzue.tech.conf` |
| Certificate | Let's Encrypt via certbot `--webroot -w /var/www/stats-acme` |
| Data | SQLite at `/var/lib/goatcounter/goatcounter-data/db.sqlite3` |
| Backups | `/etc/cron.daily/goatcounter-backup` → `/var/backups/goatcounter/` |
| Dashboard login | `contact@fuzue.tech`; password in `/root/goatcounter-admin-password` |

`vaz.io` already runs Apache with ~30 vhosts on :80/:443, so GoatCounter does **not**
use its own ACME/TLS (`-tls tls,rdr,acme`). It listens on loopback and Apache
terminates TLS in front of it. Port 8099 was chosen because nothing else on the host
listens on it or names it in a vhost.

## Rebuilding from scratch

### 1. DNS

`stats.fuzue.tech` → `A` → `72.14.186.189`, in the NearlyFreeSpeech panel for
`fuzue.tech`. This must resolve before requesting the certificate.

### 2. Binary

```sh
VERSION=v2.7.0
curl -sSL "https://github.com/arp242/goatcounter/releases/download/${VERSION}/goatcounter-${VERSION}-linux-amd64.gz" \
  | gunzip > goatcounter
sudo install -m 0755 goatcounter /usr/local/bin/goatcounter
rm goatcounter
```

### 3. User, data directory, database

```sh
sudo useradd --system --home-dir /var/lib/goatcounter --shell /usr/sbin/nologin goatcounter
sudo mkdir -p /var/lib/goatcounter/goatcounter-data
sudo chown -R goatcounter:goatcounter /var/lib/goatcounter
sudo chmod 750 /var/lib/goatcounter

sudo -u goatcounter goatcounter db create site \
  -vhost stats.fuzue.tech \
  -user.email contact@fuzue.tech \
  -db sqlite+/var/lib/goatcounter/goatcounter-data/db.sqlite3 \
  -createdb
```

### 4. Service

```sh
sudo cp analytics/goatcounter.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now goatcounter
```

### 5. Apache and the certificate

The vhost file contains both the `:80` and `:443` blocks, but the `:443` block will
fail `configtest` until the certificate exists. So: install a `:80`-only version
first, get the certificate, then install the full file.

```sh
sudo mkdir -p /var/www/stats-acme
# install the :80 block only, then:
sudo a2ensite stats.fuzue.tech.conf
sudo apache2ctl configtest && sudo systemctl reload apache2

sudo certbot certonly --webroot -w /var/www/stats-acme -d stats.fuzue.tech \
  --non-interactive --agree-tos -m contact@fuzue.tech --no-eff-email

# now install the full file from analytics/apache-stats.fuzue.tech.conf
sudo apache2ctl configtest && sudo systemctl reload apache2
```

Apache caches certificates at startup, so renewal needs a reload. The deploy hook
`/etc/letsencrypt/renewal-hooks/deploy/reload-apache-stats` does that, scoped to this
lineage:

```sh
#!/bin/sh
[ "$RENEWED_LINEAGE" = "/etc/letsencrypt/live/stats.fuzue.tech" ] || exit 0
systemctl reload apache2
```

Verify with `sudo certbot renew --cert-name stats.fuzue.tech --dry-run`.

## Backups

The whole dataset is one SQLite file; `/etc/cron.daily/goatcounter-backup` snapshots
it nightly and keeps 30 days. It uses `sqlite3 .backup` rather than `cp`, which is the
only form that is safe while the service is writing.

## Upgrading

Replace the binary and restart; `-automigrate` in the unit runs pending migrations on
startup.

```sh
sudo systemctl stop goatcounter
# repeat step 2 with the new version
sudo systemctl start goatcounter
```

## What the site sends

The snippet counts manually (`no_onload`) rather than using GoatCounter's automatic
on-load counting, so that each pageview carries the language actually rendered:

- **Pageview** on load, path suffixed with the active language — `/?lang=en`,
  `/?lang=it`, `/?lang=pt`, `/?lang=pl`. This answers whether the translations are
  being read at all.
- **Event** `lang-switch-<lang>` when a visitor changes language from the switcher,
  so a switch is not double-counted as a second pageview.

`fuzue.it` and `fuzue.pl` serve the same page and report into the same site; the
language in the path is what distinguishes them.

If the instance is down or the script is blocked, the page is unaffected — the snippet
is `async` and nothing else depends on it.

## Gotchas found while testing

- `count.js` refuses to count on `localhost`, `127.*`, and other private ranges
  (`allow_local` overrides this). Testing from a local web server records nothing;
  map a fake public hostname instead.
- GoatCounter drops hits whose User-Agent looks like a bot, and `HeadlessChrome` is
  one. Browser-automated smoke tests need a realistic `--user-agent` or they silently
  record nothing while returning HTTP 200.
- The dashboard streams data over a WebSocket at `/loader`; its `ProxyPass` must come
  before the catch-all, or the dashboard hangs. Confirmed working — a handshake
  through Apache returns `101 Switching Protocols`.
