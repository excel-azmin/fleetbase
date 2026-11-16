# Fleetbase Self-Hosted Install Guide (Docker, LAN Access)

A complete runbook for installing Fleetbase with Docker Compose on an Ubuntu server and accessing it from other machines on your network. It includes the fixes for problems commonly hit during setup (CORS errors, 502 Bad Gateway, crash-looping app container, half-finished database).

Replace `192.168.10.195` throughout with your server's IP address. To serve Fleetbase on a public domain behind a reverse proxy with HTTPS (for example Nginx Proxy Manager), follow section 7 after the base install, and section 8 to send real email over SMTP.

---

## 1. Install Docker (fresh Ubuntu 22.04+)

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
docker compose version      # confirm Compose v2 is available
```

The `usermod` step prevents the `permission denied ... /var/run/docker.sock` error. Log out and back in (or run `newgrp docker`) so it takes effect. After that you do not need `sudo` for Docker commands.

---

## 2. Clone the repository

```bash
git clone https://github.com/fleetbase/fleetbase.git
cd fleetbase
```

---

## 3. Prepare config files BEFORE the first start

### 3.1 Create `api/.env` as a file

Compose bind-mounts `./api/.env`. If that file does not exist when the stack first starts, Docker creates an empty **directory** in its place. Laravel then has no `.env` or `APP_KEY`, the app container crash-loops (`worker script has failed on restart`), nginx returns `502 Bad Gateway`, and the browser reports it as a misleading **CORS error**.

```bash
cp api/.env.example api/.env 2>/dev/null || touch api/.env
ls -la api/.env             # must start with "-rw", NOT "drwx"
```

Make sure it has an `APP_KEY`:

```bash
grep APP_KEY api/.env
# If empty, generate one:
KEY=$(docker compose run --rm --no-deps -T --entrypoint "" application php artisan key:generate --show | tail -n1 | tr -d '\r')
grep -q '^APP_KEY=' api/.env && sed -i "s|^APP_KEY=.*|APP_KEY=$KEY|" api/.env || echo "APP_KEY=$KEY" >> api/.env
```

Back up this file. The `APP_KEY` cannot be recovered if lost, and a new key cannot decrypt data encrypted with the old one.

If the repo ships `scripts/docker-install.sh`, you can run it instead. It writes the project `.env` with the socket auth settings the compose file references.

### 3.2 Create `docker-compose.override.yml`

Compose merges this file automatically, and its values take priority over `docker-compose.yml` and `api/.env`.

```yaml
services:
  application:
    environment:
      APP_URL: "http://192.168.10.195:8000"
      CONSOLE_HOST: "http://192.168.10.195:4200"
      SESSION_DOMAIN: "192.168.10.195"
      FRONTEND_HOSTS: "http://192.168.10.195:4200,192.168.10.195:4200"
      MAIL_MAILER: "log"          # emails are only written to the log; see section 8 for real SMTP

  socket:
    environment:
      SOCKETCLUSTER_OPTIONS: '{"origins":"http://192.168.10.195:*,ws://192.168.10.195:*,http://localhost:*"}'
```

### 3.3 Edit `console/fleetbase.config.json`

```json
{
  "API_HOST": "http://192.168.10.195:8000",
  "SOCKETCLUSTER_HOST": "192.168.10.195",
  "SOCKETCLUSTER_PORT": "38000",
  "SOCKETCLUSTER_SECURE": false
}
```

`localhost` in these files refers to the **browser's** machine, not the server. Always use the server's IP or domain. Give the server a static IP or DHCP reservation so it does not change after a reboot.

---

## 4. Start the stack

```bash
docker compose up -d
docker compose ps
```

Wait until `database`, `cache`, `queue` and `application` show as healthy or up.

> **Important rule for the rest of this guide:** whenever the `application` container is recreated (`up -d --force-recreate application`, an image update, or a config change), restart `httpd` right afterwards:
>
> ```bash
> docker compose restart httpd console
> ```
>
> The `httpd` container (nginx) forwards API traffic to the `application` container and keeps its old internal address after a recreate. Until `httpd` is restarted it answers `502 Bad Gateway`, and the browser shows that as a misleading **CORS error**. See section 10.1.

---

## 5. Initialize the database (first time only)

```bash
docker compose exec application bash -c "./deploy.sh"
docker compose restart application queue scheduler
```

`deploy.sh` runs migrations and seeds the data.

### If it fails with `Table 'users' already exists`

A previous install left a half-built database. Check what is there first:

```bash
docker compose exec database mysql -uroot fleetbase -e "SELECT COUNT(*) AS tables_count FROM information_schema.tables WHERE table_schema='fleetbase';"
docker compose exec database mysql -uroot fleetbase -e "SELECT COUNT(*) AS users_count FROM users;"
```

If there is no data you need, back up and reset (this **permanently deletes** the database contents):

```bash
docker compose exec -T database mysqldump -uroot fleetbase > fleetbase-backup.sql
docker compose exec database mysql -uroot -e "DROP DATABASE fleetbase; CREATE DATABASE fleetbase CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
docker compose exec application bash -c "./deploy.sh"
docker compose restart application queue scheduler
```

If the database holds real data, do not drop it. Inspect `php artisan migrate:status` and reconcile the `migrations` table instead.

---

## 6. Open it

| What | URL |
|---|---|
| Console | `http://192.168.10.195:4200` |
| API | `http://192.168.10.195:8000` |
| Websocket | `ws://192.168.10.195:38000` |

Verify the API from any machine:

```bash
curl -i -H "Origin: http://192.168.10.195:4200" http://192.168.10.195:8000/int/v1/onboard/should-onboard
```

A `200` response with an `Access-Control-Allow-Origin` header means it is working. Open the console and create the admin account on the onboarding screen. Hard-refresh with Ctrl+Shift+R if the page is stale.

Open the ports on the firewall if needed:

```bash
sudo ufw allow 4200,8000,38000/tcp
```

---

## 7. Run on a domain with HTTPS (reverse proxy)

Use this when you put Fleetbase behind Nginx Proxy Manager (NPM), Caddy, or another reverse proxy. The examples use `avolytic.com`; replace it with your domain.

### 7.1 Why the LAN config breaks on a domain

If the page is served over HTTPS but the console still calls `http://192.168.10.195:8000`, the browser blocks it as **Mixed Content**. The LAN address is also unreachable from outside your network. The browser then reports a misleading **CORS error**. The fix is to give the console, the API and the websocket each their own public HTTPS address, and point the config at those.

### 7.2 DNS

Create A records pointing at your public IP (or a CNAME to your main host):

| Hostname | Purpose |
|---|---|
| `fleetbase.avolytic.com` | Console (web UI) |
| `fleetbase-api.avolytic.com` | API |
| `fleetbase-socket.avolytic.com` | Websocket |

Forward ports 80 and 443 on your router to the proxy server.

### 7.3 Proxy hosts (Nginx Proxy Manager)

Create three proxy hosts:

| Domain | Scheme | Forward host | Port | Websockets support |
|---|---|---|---|---|
| `fleetbase.avolytic.com` | http | 192.168.10.195 | 4200 | off |
| `fleetbase-api.avolytic.com` | http | 192.168.10.195 | 8000 | on |
| `fleetbase-socket.avolytic.com` | http | 192.168.10.195 | 38000 | **on (required)** |

For each host, open the SSL tab, request a Let's Encrypt certificate, and enable **Force SSL**. Leave the Advanced tab empty. Do not add your own CORS headers there, because duplicate headers break the preflight check.

If you use plain Caddy instead of NPM, the equivalent is:

```
fleetbase.avolytic.com {
    reverse_proxy 192.168.10.195:4200
}
fleetbase-api.avolytic.com {
    reverse_proxy 192.168.10.195:8000
}
fleetbase-socket.avolytic.com {
    reverse_proxy 192.168.10.195:38000
}
```

### 7.4 Console config

Edit `console/fleetbase.config.json`:

```json
{
  "API_HOST": "https://fleetbase-api.avolytic.com",
  "SOCKETCLUSTER_HOST": "fleetbase-socket.avolytic.com",
  "SOCKETCLUSTER_PORT": "443",
  "SOCKETCLUSTER_SECURE": true
}
```

### 7.5 Override file

Replace the LAN values in `docker-compose.override.yml`:

```yaml
services:
  application:
    environment:
      APP_URL: "https://fleetbase-api.avolytic.com"
      CONSOLE_HOST: "https://fleetbase.avolytic.com"
      SESSION_DOMAIN: ".avolytic.com"
      FRONTEND_HOSTS: "https://fleetbase.avolytic.com,fleetbase.avolytic.com"
      MAIL_MAILER: "log"

  socket:
    environment:
      SOCKETCLUSTER_OPTIONS: '{"origins":"https://fleetbase.avolytic.com:*,wss://fleetbase.avolytic.com:*"}'
```

This example still has `MAIL_MAILER: "log"`, so emails are only written to the log. For real email, use the combined override file in section 8.2 instead.

Notes:

- `FRONTEND_HOSTS` entries must match the browser origin exactly: include `https://` and no trailing slash.
- The console reads a single `API_HOST`, so you cannot use the LAN address and the domain from the same console build at the same time. To keep LAN access too, also add `http://192.168.10.195:4200` to `FRONTEND_HOSTS` and the socket origins, and reach the console through the domain.

### 7.6 Apply

```bash
docker compose up -d --force-recreate application socket scheduler queue
docker compose exec application php artisan config:clear
docker compose exec application php artisan cache:clear
docker compose restart httpd console      # required, otherwise the API returns 502 (shown as CORS)
sleep 20
docker compose ps                         # application should be healthy
```

### 7.7 Test

```bash
curl -i -H "Origin: https://fleetbase.avolytic.com" https://fleetbase-api.avolytic.com/int/v1/onboard/should-onboard
```

You should see `200` and `Access-Control-Allow-Origin: https://fleetbase.avolytic.com`. Then hard-refresh the console with Ctrl+Shift+R. In the browser's Network tab, `should-onboard` should call `https://fleetbase-api.avolytic.com/...`.

If it still calls `192.168.10.195`, open `https://fleetbase.avolytic.com/fleetbase.config.json` in the browser to see which config the console is actually serving.

### 7.8 Domain troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Mixed Content` error in the browser console | `API_HOST` still uses `http://` or the LAN IP | Section 7.4, then `docker compose restart console` |
| CORS error after the Mixed Content fix | Origin not in `FRONTEND_HOSTS` | Match it exactly, including `https://` |
| `502` from the API hostname | Wrong forward port or scheme in the proxy | Forward to `http` on port 8000 |
| Duplicate `Access-Control-Allow-Origin` header | CORS headers added in the proxy Advanced tab | Remove them and let the API set CORS |
| Websocket errors / live tracking missing | Websockets Support off on the socket host, or wrong port | Turn it on, use port 443 with `SOCKETCLUSTER_SECURE: true` |
| Certificate request fails | DNS not pointing at the proxy, or ports 80/443 not forwarded | Check the A records and router forwarding |
| Login works but session is lost | `SESSION_DOMAIN` mismatch | Use `.avolytic.com` so API and console share it |

---

## 8. Configure SMTP email

Fleetbase sends invitations, password resets and notifications by email. The override file above sets `MAIL_MAILER: "log"`, which writes emails to the log instead of sending them. To send real mail, switch to SMTP.

### 8.1 Keep the password out of the YAML

Docker Compose automatically reads a file named `.env` in the project folder (next to `docker-compose.yml`) and substitutes `${VARIABLE}` values. Put the credentials there:

```bash
cd /home/excel/fleetbase
nano .env
```

If the file already exists (the install script may have created it with `SOCKETCLUSTER_*` values), append to it:

```
MAIL_HOST=smtp.example.com
MAIL_PORT=587
MAIL_USERNAME=postmaster@example.com
MAIL_PASSWORD=your-smtp-password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=noreply@avolytic.com
```

```bash
chmod 600 .env
```

This `.env` is the Compose project file. It is different from `api/.env`, which is the Laravel file inside the container.

### 8.2 Complete `docker-compose.override.yml` (domain + SMTP)

Location: `/home/excel/fleetbase/docker-compose.override.yml`. This replaces the earlier override files. It contains the domain settings from section 7 and the SMTP settings. The `queue` and `scheduler` services need the mail variables too, because emails are sent by the queue worker. The `x-mail-env` block is defined once and reused by all three services.

```yaml
# Shared SMTP settings, reused by application, queue and scheduler
x-mail-env: &mail-env
  MAIL_MAILER: "smtp"
  MAIL_HOST: "${MAIL_HOST}"
  MAIL_PORT: "${MAIL_PORT}"
  MAIL_USERNAME: "${MAIL_USERNAME}"
  MAIL_PASSWORD: "${MAIL_PASSWORD}"
  MAIL_ENCRYPTION: "${MAIL_ENCRYPTION}"
  MAIL_FROM_ADDRESS: "${MAIL_FROM_ADDRESS}"
  MAIL_FROM_NAME: "Fleetbase"

services:
  application:
    environment:
      <<: *mail-env
      APP_URL: "https://fleetbase-api.avolytic.com"
      CONSOLE_HOST: "https://fleetbase.avolytic.com"
      SESSION_DOMAIN: ".avolytic.com"
      FRONTEND_HOSTS: "https://fleetbase.avolytic.com,fleetbase.avolytic.com"

  queue:
    environment:
      <<: *mail-env

  scheduler:
    environment:
      <<: *mail-env

  socket:
    environment:
      SOCKETCLUSTER_OPTIONS: '{"origins":"https://fleetbase.avolytic.com:*,wss://fleetbase.avolytic.com:*"}'
```

If you edit this file and accidentally drop the `APP_URL`, `CONSOLE_HOST`, `SESSION_DOMAIN` or `FRONTEND_HOSTS` lines under `application`, the API stops allowing your console origin and you get a real CORS error. Verify with `docker compose exec application printenv | grep -E "FRONTEND_HOSTS|CONSOLE_HOST|SESSION_DOMAIN|APP_URL"`.

If your SMTP password contains `$`, write it as `$$` in the Compose `.env` file so Compose does not treat it as a variable.

### 8.3 Port and encryption settings

| Port | `MAIL_ENCRYPTION` | Notes |
|---|---|---|
| 587 | `tls` | STARTTLS. The usual choice |
| 465 | `ssl` | Implicit TLS. Do not combine port 465 with `tls` |
| 25 | empty | Unencrypted. Often blocked by ISPs and cloud hosts |

Provider notes:

- **Gmail / Google Workspace:** host `smtp.gmail.com`, port 587, and an **App Password** (needs 2-step verification on). Your normal password will not work.
- **Microsoft 365:** host `smtp.office365.com`, port 587. SMTP AUTH must be enabled for the mailbox.
- **ZeptoMail (Zoho):** host `smtp.zeptomail.com`, port 587 with `tls` (or 465 with `ssl`). The username is normally `emailapikey` and the password is the Send Mail token. `MAIL_FROM_ADDRESS` must be on a domain you have verified in ZeptoMail.
- **Transactional services** (Mailgun, SendGrid, Brevo, Amazon SES): use their SMTP host and credentials. These usually deliver better than a personal mailbox.
- `MAIL_FROM_ADDRESS` should be an address your SMTP provider allows you to send as, otherwise the message is rejected or lands in spam.

### 8.4 Apply

```bash
docker compose config | grep -E "MAIL_(MAILER|HOST|PORT|ENCRYPTION|FROM)"   # confirm values merged
docker compose up -d --force-recreate application queue scheduler
docker compose exec application php artisan config:clear
docker compose exec application php artisan cache:clear
docker compose restart httpd console queue   # httpd restart is required after recreating application
sleep 20
docker compose ps                            # application should be healthy
```

### 8.5 Send a test email

```bash
docker compose exec application php artisan tinker --execute='Mail::raw("Fleetbase SMTP test", fn($m) => $m->to("you@example.com")->subject("Fleetbase SMTP test"));'
```

If it prints nothing and the email arrives, SMTP works. Any connection or authentication error is printed straight to the terminal. You can also confirm the active mailer with:

```bash
docker compose exec application php artisan about | grep -i mail
```

It should say `smtp`. A value of `log` or `ses` means the override has not been applied.

### 8.6 SMTP troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `about` still shows `log` or `ses` | Override not applied, or `MAIL_MAILER` set elsewhere | Check `docker compose config`, recreate containers, clear config |
| `Connection could not be established` / timeout | Wrong host or port, or outbound SMTP blocked by firewall or ISP | Test with `nc -vz smtp.example.com 587`. Try 587 instead of 25 |
| `Authentication failed` / `535` | Wrong credentials, or a normal password used where an app password is required | Use an app password or a dedicated SMTP user |
| `certificate verify failed` | Wrong `MAIL_ENCRYPTION` for the port | 587 uses `tls`, 465 uses `ssl` |
| Test works but app emails never arrive | The `queue` container lacks the SMTP variables or was not restarted | Add them to `queue` (8.2) and run `docker compose restart queue` |
| Emails arrive in spam | Sender domain has no SPF, DKIM or DMARC | Add the DNS records your SMTP provider gives you |
| Failed jobs after a mail error | Queue retries | `docker compose exec application php artisan queue:failed`, fix the config, then `php artisan queue:retry all` |

Some Fleetbase versions also let admins edit mail settings in the console under the admin settings area. Values in the container environment take priority, so if you use that screen, remove the `MAIL_*` lines from the override.

### 8.7 Complete file reference (domain + SMTP)

All the files for a domain setup with SMTP, in one place. Replace `avolytic.com`, the IP and the SMTP values with your own.

**1. `docker-compose.override.yml`**: see section 8.2.

**2. `.env`** (Compose project file, `/home/excel/fleetbase/.env`):

```env
# --- SMTP ---
MAIL_HOST=smtp.example.com
MAIL_PORT=587
MAIL_USERNAME=postmaster@example.com
MAIL_PASSWORD=your-smtp-password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=noreply@avolytic.com

# --- Socket (keep existing values if scripts/docker-install.sh already created them) ---
SOCKETCLUSTER_AUTH_ENABLED=false
SOCKETCLUSTER_AUTH_KEY=
SOCKETCLUSTER_AUTH_MODE=
SOCKETCLUSTER_ORIGIN=https://fleetbase.avolytic.com
```

```bash
chmod 600 .env
```

**3. `console/fleetbase.config.json`:**

```json
{
  "API_HOST": "https://fleetbase-api.avolytic.com",
  "SOCKETCLUSTER_HOST": "fleetbase-socket.avolytic.com",
  "SOCKETCLUSTER_PORT": "443",
  "SOCKETCLUSTER_SECURE": true
}
```

**4. `api/.env`** (Laravel file mounted into the container; must be a file, not a directory):

```env
APP_KEY=base64:PASTE_YOUR_EXISTING_KEY_HERE
```

Keep the `APP_KEY` you already have. Everything else comes from the override and compose file, which take priority over this file.

**5. Nginx Proxy Manager hosts:** see section 7.3.

**Apply everything:**

```bash
cd /home/excel/fleetbase
docker compose config | grep -E "MAIL_(MAILER|HOST|PORT|ENCRYPTION|FROM_ADDRESS)|FRONTEND_HOSTS|CONSOLE_HOST"
docker compose up -d --force-recreate application queue scheduler socket
docker compose exec application php artisan config:clear
docker compose exec application php artisan cache:clear
docker compose restart httpd console queue
sleep 20
docker compose ps
curl -i -H "Origin: https://fleetbase.avolytic.com" https://fleetbase-api.avolytic.com/int/v1/onboard/should-onboard
```

---

## 9. Day-to-day operations

```bash
docker compose ps                                   # status
docker compose logs -f --tail=100 application       # follow logs
docker compose restart <service>                    # restart one service
docker compose down                                 # stop (data kept in docker/database/mysql)
docker compose up -d                                # start
```

### Apply override or config changes

```bash
docker compose up -d --force-recreate application queue scheduler socket
docker compose exec application php artisan config:clear
docker compose exec application php artisan cache:clear
docker compose restart httpd console queue   # always restart httpd after recreating application
```

### Update

```bash
git pull
docker compose pull
docker compose up -d
docker compose restart httpd console
docker compose exec application ./deploy.sh        # run new migrations
```

### Backup

```bash
docker compose exec -T database mysqldump -uroot fleetbase > fleetbase-$(date +%F).sql
```

Also back up `api/.env` and `api/storage/app`.

---

## 10. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Browser "CORS error" on `should-onboard` | Usually the API returns 502, or the origin is not allowed | Follow section 10.1 |
| CORS error right after recreating `application` | `httpd` still points at the old container | `docker compose restart httpd console` |
| `502 Bad Gateway` from `:8000` | Application container crash-looping | `docker compose ps`, then check `api/.env` |
| `worker script has failed on restart` loop | `api/.env` missing, is a directory, or has no `APP_KEY` | Section 3.1 |
| `permission denied ... docker.sock` | User not in the docker group | `sudo usermod -aG docker $USER`, then re-login |
| "Fleetbase is not installed or configured" | Migrations or seed did not complete | Section 5 |
| `Table 'users' already exists` | Half-finished earlier install | Reset the DB (section 5) |
| Override changes not applied | Containers not recreated | Section 9, apply changes |
| `scheduler` shows unhealthy | Same cause as the app crash | Fix the app first, then it recovers |
| Console loads but calls `localhost` | `API_HOST` wrong in `fleetbase.config.json` | Section 3.3, then `docker compose restart console` |
| Mixed Content or CORS error on a domain | Console still points at the LAN IP | Section 7 |
| Invitation or reset emails never arrive | Mailer is still `log`, or SMTP not set on `queue` | Section 8 |

### 10.1 CORS error: step-by-step fix

The browser prints "blocked by CORS policy" for several unrelated problems. Work through these in order.

**Step 1. Restart `httpd` and `console`.** This is the most common cause after you recreate `application` (for example after editing the override or SMTP settings):

```bash
cd /home/excel/fleetbase
docker compose restart httpd console
sleep 20
docker compose ps
```

`application` should show `healthy`. It takes 15 to 30 seconds after a recreate.

**Step 2. Test the API directly:**

```bash
curl -i -H "Origin: https://fleetbase.avolytic.com" https://fleetbase-api.avolytic.com/int/v1/onboard/should-onboard
```

| Result | Meaning | Fix |
|---|---|---|
| `200` and `Access-Control-Allow-Origin: https://fleetbase.avolytic.com` | API is fine | Hard-refresh the browser with Ctrl+Shift+R. Check the Network tab to make sure the console calls the API hostname |
| `502 Bad Gateway` | `httpd` cannot reach the app, or the app is crash-looping | Repeat step 1. If it persists: `docker compose logs --tail=40 application \| grep -v "worker script"`, and check `api/.env` is a file with an `APP_KEY` |
| `200` but no `Access-Control-Allow-Origin` header | Origin not allowed, or the override lost its settings | Step 3 |
| Cannot connect / certificate error | DNS, proxy host or certificate problem | Check the Nginx Proxy Manager host and SSL settings |

**Step 3. Check the origin settings the container actually sees:**

```bash
docker compose exec application printenv | grep -E "FRONTEND_HOSTS|CONSOLE_HOST|SESSION_DOMAIN|APP_URL"
```

They must show the `https://fleetbase...` values from section 8.2. If they are missing, the `application` block of your override lost them. Restore them, then recreate `application`, clear the config cache and restart `httpd` (section 8.4). The `FRONTEND_HOSTS` entry must match the browser origin exactly: `https://` included, no trailing slash.

**Step 4. Check the console config.** Open `https://fleetbase.avolytic.com/fleetbase.config.json` in the browser. `API_HOST` must be `https://fleetbase-api.avolytic.com`. If it shows `http://` or the LAN IP, you get a Mixed Content error followed by a CORS error. Fix `console/fleetbase.config.json` and run `docker compose restart console`.

**Step 5. Remove duplicate CORS headers.** If the proxy's Advanced tab contains `add_header Access-Control-Allow-Origin` lines, delete them. The API sets CORS itself, and two headers make the browser reject the response.

Useful diagnostics:

```bash
df -h                                               # a full disk breaks MySQL and the app
docker compose exec application printenv | grep -E "FRONTEND_HOSTS|CONSOLE_HOST|SESSION_DOMAIN"
docker compose run --rm --no-deps --entrypoint "" application php artisan about
```

---

## 11. Security notes

- The default compose file sets `MYSQL_ALLOW_EMPTY_PASSWORD: "yes"` and publishes MySQL on port 3306. This is acceptable on an isolated test machine only. For anything shared, remove the `3306:3306` port mapping and set a real password.
- Use a dedicated SMTP user or app password, and keep it in the Compose `.env` file (`chmod 600`), not in the YAML.
- For production, serve over HTTPS with a domain (section 7) and set `SOCKETCLUSTER_AUTH_ENABLED=true` with a secret key.
- The compose file disables MySQL binary logging (`--skip-log-bin`) because telematics ingest made it grow past 300 GB. Do not enable replication or point-in-time recovery without turning it back on.
- When exposing Fleetbase publicly, do not forward ports 3306 (MySQL), 4200, 8000 or 38000 on the router. Only expose 80 and 443 on the proxy.
- Keep `api/.env` out of version control and restrict its permissions.
