# Deploying Smart Incident Management with Docker

Runs the whole stack with one command: PostgreSQL, the Spring Boot backend, the React frontend (served by nginx), plus Prometheus and Grafana for monitoring.

```
Browser ──> nginx :80 ──┬─ /        React app (static files)
                        └─ /api/    Spring Boot :8080 ──> PostgreSQL
                                          ▲
Prometheus ── scrapes /actuator/prometheus ┘   (internal network only)
Grafana ── reads Prometheus
```

Only nginx is open to the internet. The database and backend are not published. Grafana and Prometheus are bound to `127.0.0.1` by default.

## Files added

| File | Purpose |
|---|---|
| `docker-compose.yml` | All 6 services, volumes, healthchecks |
| `.env.example` | Settings template (copy to `.env`) |
| `backend/Dockerfile`, `backend/.dockerignore` | Builds the Spring Boot jar, runs on Java 21 |
| `frontend/Dockerfile`, `frontend/.dockerignore` | Builds the React app, serves it with nginx |
| `frontend/nginx.conf` | Serves the SPA, proxies `/api`, blocks `/actuator` |
| `monitoring/prometheus/prometheus.yml`, `alerts.yml` | Scrape targets and 6 alert rules |
| `monitoring/grafana/...` | Auto-provisioned datasource and the "Smart IMS Overview" dashboard |

## Read this first (3 things about the project itself)

1. **Rotate the credentials in `.env.properties`.** That file is committed to the repository and contains what look like a real Gmail app password and Mailtrap credentials. Anyone who can see the repo (and its git history) has them. Revoke that Gmail app password, create a new one, and remove the file from git (`git rm --cached .env.properties`). The Docker setup does not use that file.
2. **Public registration creates `SUPER_ADMIN` accounts.** `POST /api/auth/register` is open to anyone and assigns the highest role after an email OTP. Create your own account first, then block registration: uncomment the `/api/auth/register` block in `frontend/nginx.conf` and run `docker compose up -d --build frontend`. (Fixing it properly means a change in the Java code.)
3. **Email must work to create the first account.** There is no seeded admin; the first user signs up through the Register page and receives an OTP by email. Set `SMTP_USERNAME` / `SMTP_PASSWORD` in `.env` (for Gmail, use an App Password).

## Deploy

You need a Linux server (or your own machine) with Docker Engine and the Compose plugin, at least 2 GB RAM (building the backend uses about 1 GB; add swap if you have less), and port 80 open (plus 443 if you add HTTPS).

```bash
# 1. Install Docker (skip if already installed)
curl -fsSL https://get.docker.com | sh

# 2. Get the code and drop the new files into the repo root
git clone https://github.com/Drtrivedi05/smart-incident-management.git
cd smart-incident-management
# copy the files from this package here (same folder layout)

# 3. Configure
cp .env.example .env
nano .env        # set PUBLIC_URL, passwords, JWT_SECRET, SMTP_*
                 # JWT_SECRET: openssl rand -base64 48

# 4. Build and start (first build takes several minutes)
docker compose up -d --build

# 5. Check
docker compose ps          # all services "running"/"healthy"
docker compose logs -f backend
```

`PUBLIC_URL` must be exactly what users type in the browser, for example `http://203.0.113.10` or `https://ims.example.com`. It is baked into the frontend at build time, so if you change it run `docker compose up -d --build frontend`.

Open `PUBLIC_URL` -> Register -> enter the OTP from your email -> log in.

## Monitoring

Grafana and Prometheus only listen on the server itself. From your own computer, open an SSH tunnel:

```bash
ssh -L 3000:127.0.0.1:3000 -L 9090:127.0.0.1:9090 youruser@your-server
```

- Grafana: http://localhost:3000, user `admin`, password = `GRAFANA_ADMIN_PASSWORD`. The "Smart IMS Overview" dashboard is the home page (backend/Postgres status, request rate, 5xx %, latency, JVM heap, CPU, DB pool).
- Prometheus: http://localhost:9090/targets (all three targets should be UP) and `/alerts`.

The alert rules (backend down, Postgres down, 5xx > 5%, latency > 1s, heap > 90%, DB pool waiting) are evaluated and shown in Prometheus, but nothing is *sent* anywhere because there is no Alertmanager. Adding one is the natural next step if you want email/Slack notifications.

## Everyday operations

```bash
docker compose logs -f backend          # app logs
docker compose up -d --build            # after git pull / config changes
docker compose restart backend
docker compose down                     # stop (data is kept)
# docker compose down -v  <-- also DELETES the database and uploads. Avoid.

# Backup the database
docker compose exec -T postgres pg_dump -U smartims smart_ims > backup_$(date +%F).sql
# Restore
docker compose exec -T postgres psql -U smartims smart_ims < backup.sql
```

Uploaded attachments live in the `uploads` Docker volume; back it up too if attachments matter.

## HTTPS

On plain HTTP, passwords and login tokens travel unencrypted. For anything beyond a demo, put HTTPS in front: with a domain name, the easiest options are Caddy or Cloudflare in front of port 80. Then set `PUBLIC_URL=https://your-domain` and rebuild the frontend.

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `Set PUBLIC_URL in .env` (or similar) when starting | A required value in `.env` is empty |
| Backend stays "unhealthy" | `docker compose logs backend`: usually a wrong DB password or a `JWT_SECRET` shorter than 32 characters. If you changed `POSTGRES_PASSWORD` after the first start, the old one is still stored in the `pgdata` volume. |
| Browser shows network errors to `localhost:8080` | The frontend was built with the wrong `PUBLIC_URL`. Fix `.env`, then `docker compose up -d --build frontend`. |
| No OTP email | Check `SMTP_*`; look for mail errors in `docker compose logs backend`. Gmail needs an App Password. |
| Upload fails | Limit is 10 MB (set in compose and nginx). Raise both together. |
| Prometheus target `smartims-backend` is DOWN | Backend not healthy yet, or it crashed: check backend logs. |

## What has and hasn't been tested

Checked: `docker compose config` renders correctly with all variables; `promtool` accepts `prometheus.yml`, all 6 alert rules and all 19 dashboard queries.

Not run (the environment these were written in has no Docker daemon and blocks Maven Central, Docker Hub and npm downloads): the actual image builds and the running stack. Expect to possibly adjust small things on your first `docker compose up`. The `grafana/grafana:12.1.0` and `postgres-exporter:v0.17.1` image tags could not be confirmed; if a pull fails, change the tag in `docker-compose.yml`.
