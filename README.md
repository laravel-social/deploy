# Laravel Social - deployment configs

Deploy configs for [Laravel Social](https://laravelsocial.com), distributed as a public Docker image:
`docker.io/izdrail/app.laravelsocial.com:latest`

This repo contains hosting-provider configuration only. No application code.

> **Status:** these configs are written against each provider's current docs and the app's actual Docker image, but they have **not been end-to-end deployed or tested** by the project team. Treat first deploy as a dry run.

## One container, everything inside

The image starts web (Laravel Octane on port **13001**), the queue worker (Horizon), the task scheduler, and Redis together via supervisord. **Do not add provider worker services, cron jobs, or scheduler add-ons** - they would run the same scheduled jobs twice.

## Required manual setup (all providers)

No deploy button or config here gives you a fully configured app. Complete these steps during or immediately after deploy:

1. **APP_KEY - the app will not boot without it.** Generate once locally: `php artisan key:generate --show` (or `openssl rand -base64 32` prefixed with `base64:`), set as an env var/secret. Example of a valid format: `base64:9hK3...==` (32 random bytes, base64-encoded).
2. **APP_URL** - your public https URL, e.g. `https://laravel-social.onrender.com`.
3. **Migrate once** after first deploy: run `php artisan migrate --force` in the provider's console/shell. Nothing auto-migrates.
4. **Persistence - REQUIRED, not optional.** The app defaults to SQLite (stored at `/var/www/database/database.sqlite`) and local-disk uploads (`/var/www/storage`). On every platform here the container filesystem is ephemeral or reset on redeploy, so without one of the setups below **you lose your database and uploads**:
   - **Database:** attach an external MySQL (set `DB_CONNECTION=mysql`, `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`) **or** a persistent volume mounted at `/var/www/database` (Render and Fly configs below include this; DigitalOcean App Platform has no volumes, so MySQL is required there).
   - **Uploads/media:** set `FILESYSTEM_DISK=s3` plus `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`, `AWS_BUCKET` (and `AWS_ENDPOINT`/`AWS_URL` for S3-compatible stores like DigitalOcean Spaces or Cloudflare R2) **or** mount a volume at `/var/www/storage`.

### Example: external MySQL + S3-compatible storage (works on every provider)

```
DB_CONNECTION=mysql
DB_HOST=your-mysql-host.example.com
DB_PORT=3306
DB_DATABASE=laravel_social
DB_USERNAME=laravel_social
DB_PASSWORD=<your database password>

FILESYSTEM_DISK=s3
AWS_ACCESS_KEY_ID=<your spaces/r2 key id>
AWS_SECRET_ACCESS_KEY=<your spaces/r2 secret>
AWS_DEFAULT_REGION=fra1
AWS_BUCKET=my-laravel-social-uploads
AWS_ENDPOINT=https://fra1.digitaloceanspaces.com   # omit for real AWS S3
AWS_URL=https://my-laravel-social-uploads.fra1.digitaloceanspaces.com
```

## Providers

| Provider | How | Config here |
| --- | --- | --- |
| Koyeb | One-click button (no config file needed; port 13001 and env vars are still set on the service): [deploy](https://app.koyeb.com/deploy?type=docker&name=laravel-social&image=docker.io/izdrail/app.laravelsocial.com:latest), then set port 13001 + env | - |
| Render | [Deploy to Render](https://render.com/deploy?repo=https://github.com/laravel-social/deploy) (blueprint: web service + SQLite disk for the database; uploads still require S3 - Render allows only one disk per service) | `render.yaml` |
| DigitalOcean | [Deploy to App Platform](https://cloud.digitalocean.com/apps/new?repo=https://github.com/laravel-social/deploy/tree/main) (add managed MySQL + Spaces before real use) | `.do/deploy.template.yaml` |
| Fly.io | `fly deploy --image izdrail/app.laravelsocial.com:latest` with the included `fly.toml` (create the volume first) | `fly.toml` |
| Railway | Guide only: new project -> Docker image -> `izdrail/app.laravelsocial.com`, port 13001, attach a volume at `/var/www/database` | - |

Heroku is intentionally absent: its container runtime requires the app to listen on the injected `$PORT` and ignores `EXPOSE`, while this image binds a fixed 13001 - the image cannot run there as shipped.

---

## Worked setup examples

### Koyeb (one click + 3 settings)

1. Click the deploy link above and sign in. The form is prefilled with the image `docker.io/izdrail/app.laravelsocial.com:latest`.
2. In the service settings set the **port to 13001** (HTTP).
3. Add environment variables:
   ```
   APP_KEY=base64:<your generated key>
   APP_URL=https://laravel-social-<your-org>.koyeb.app
   ```
4. Deploy, then from the Koyeb console run once: `php artisan migrate --force`.
5. Before real use: create a Koyeb volume mounted at `/var/www/database` (or use external MySQL) and configure S3 for uploads (example block above). Without this, every redeploy wipes the database and uploads.

### Render (blueprint button)

1. Click "Deploy to Render". Render reads `render.yaml` and prompts for `APP_KEY` and `APP_URL` - paste the generated key and use `https://<service-name>.onrender.com`.
2. The blueprint creates one web service (a paid plan - persistent disks require one; check Render's current pricing at checkout) with `PORT=13001` preset and a 1GB disk mounted at `/var/www/database` - the default SQLite database survives redeploys.
3. After first deploy, open the Render Shell and run once: `php artisan migrate --force`.
4. Uploads are NOT covered by that disk (Render allows one disk per service): add the S3 env block from the example above before real use.
5. Do not add Render background workers or cron jobs - the container runs the queue and scheduler itself.

### DigitalOcean App Platform (button)

1. Click "Deploy to App Platform". App Platform reads `.do/deploy.template.yaml` and prompts for `APP_KEY` (no default is shipped on purpose). `APP_URL` fills in automatically.
2. The template creates one web service from the image on port 13001 (`basic-xs`; check the current price shown in the App Platform create flow).
3. After deploy, open the console and run once: `php artisan migrate --force`.
4. App Platform has NO persistent disks. Before real use, attach a managed MySQL database and a Spaces bucket, and set the MySQL + S3 env block from the example above. Until then, every redeploy resets the database and uploads.

### Fly.io (CLI + fly.toml)

```bash
fly apps create my-laravel-social              # app names are globally unique - pick your own
fly volumes create laravel_social_data --size 1 --region ams
fly secrets set APP_KEY="base64:<your generated key>" APP_URL="https://my-laravel-social.fly.dev"
fly deploy --image izdrail/app.laravelsocial.com:latest
fly ssh console -C "php artisan migrate --force"   # one time
```

The included `fly.toml` already sets `internal_port = 13001` and mounts the volume at `/var/www/database` for the default SQLite database. Add the S3 env block for uploads before real use (`fly secrets set FILESYSTEM_DISK=s3 AWS_...`).

### Railway (guide only)

1. New project -> Deploy from Docker image -> `izdrail/app.laravelsocial.com:latest`.
2. Set the service port to **13001**.
3. Variables:
   ```
   APP_KEY=base64:<your generated key>
   APP_URL=https://<your-service>.up.railway.app
   ```
4. Attach a volume mounted at `/var/www/database` (database) - or add MySQL from the Railway catalog and set the MySQL env block.
5. Add the S3 env block for uploads.
6. Deploy, then from the Railway shell run once: `php artisan migrate --force`. Do not add a Railway cron or second service for the queue - the container handles it.
