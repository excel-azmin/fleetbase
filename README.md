# Fleetbase Self-Hosted Install Guide (Docker, LAN Access)

A complete runbook for installing Fleetbase with Docker Compose on an Ubuntu server and accessing it from other machines on your network. It includes the fixes for problems commonly hit during setup (CORS errors, 502 Bad Gateway, crash-looping app container, half-finished database).

Replace `192.168.10.195` throughout with your server's IP address or domain.

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
      MAIL_MAILER: "log"

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

## 7. Day-to-day operations

```bash
docker compose ps                                   # status
docker compose logs -f --tail=100 application       # follow logs
docker compose restart <service>                    # restart one service
docker compose down                                 # stop (data kept in docker/database/mysql)
docker compose up -d                                # start
```

### Apply override or config changes

```bash
docker compose up -d --force-recreate application socket
docker compose exec application php artisan config:clear
docker compose exec application php artisan cache:clear
docker compose restart console httpd
```

### Update

```bash
git pull
docker compose pull
docker compose up -d
docker compose exec application ./deploy.sh        # run new migrations
```

### Backup

```bash
docker compose exec -T database mysqldump -uroot fleetbase > fleetbase-$(date +%F).sql
```

Also back up `api/.env` and `api/storage/app`.

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Browser "CORS error" on `should-onboard` | API returning 502, or origin not allowed | `curl` the API. A 502 means the app is crashing. Otherwise check `FRONTEND_HOSTS` |
| `502 Bad Gateway` from `:8000` | Application container crash-looping | `docker compose ps`, then check `api/.env` |
| `worker script has failed on restart` loop | `api/.env` missing, is a directory, or has no `APP_KEY` | Section 3.1 |
| `permission denied ... docker.sock` | User not in the docker group | `sudo usermod -aG docker $USER`, then re-login |
| "Fleetbase is not installed or configured" | Migrations or seed did not complete | Section 5 |
| `Table 'users' already exists` | Half-finished earlier install | Reset the DB (section 5) |
| Override changes not applied | Containers not recreated | Section 7, apply changes |
| `scheduler` shows unhealthy | Same cause as the app crash | Fix the app first, then it recovers |
| Console loads but calls `localhost` | `API_HOST` wrong in `fleetbase.config.json` | Section 3.3, then `docker compose restart console` |

Useful diagnostics:

```bash
df -h                                               # a full disk breaks MySQL and the app
docker compose exec application printenv | grep -E "FRONTEND_HOSTS|CONSOLE_HOST|SESSION_DOMAIN"
docker compose run --rm --no-deps --entrypoint "" application php artisan about
```

---

## 9. Security notes

- The default compose file sets `MYSQL_ALLOW_EMPTY_PASSWORD: "yes"` and publishes MySQL on port 3306. This is acceptable on an isolated test machine only. For anything shared, remove the `3306:3306` port mapping and set a real password.
- For production, serve over HTTPS with a domain (the repo includes a `Caddyfile`) and set `SOCKETCLUSTER_AUTH_ENABLED=true` with a secret key.
- The compose file disables MySQL binary logging (`--skip-log-bin`) because telematics ingest made it grow past 300 GB. Do not enable replication or point-in-time recovery without turning it back on.
- Keep `api/.env` out of version control and restrict its permissions.
