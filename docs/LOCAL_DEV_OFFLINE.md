# Running seqtoid-web locally, offline (no CZI/UCSF AWS)

This is the verified path to **build and run the web app on a single machine without any
customer (CZI/UCSF) ECR images, AWS credentials, or SSM/chamber secrets** (CZID-235).

It relies only on changes already on `main`:
- the local docker-compose network uses the `bridge` driver (CZID-273),
- `scripts/token_auth.py` loads its key in-memory / from the mounted test PEM (CZID-272),
- the `OFFLINE=1` Makefile mode (skips `aws-oidc` / ECR login),
- the dev `web` command runs `rails server` directly (no `chamber exec`, so no SSM).

## Prerequisites
- Docker Desktop (or any single-host Docker; **Swarm not required**).
- No AWS profile / credentials needed.

> Compose interpolates `${AWS_ACCOUNT_ID}` / `${AWS_REGION}` into the local image tag.
> Offline, just give them throwaway values so the tag is valid:
> `export AWS_ACCOUNT_ID=000000000000 AWS_REGION=us-west-2`

## 1. Build the web image (no ECR pull — builds locally)
```bash
OFFLINE=1 docker compose --env-file web.env build web
# (equivalently: make local-build OFFLINE=1 — local-ecr-login no-ops without an idseq-dev profile)
```

## 2. Start the database + cache, create the schema
```bash
docker compose up -d db redis                      # MySQL 5.7 + Redis
docker exec "$(docker compose ps -q db)" \
  mysql -uroot -e "CREATE DATABASE IF NOT EXISTS idseq_development; CREATE DATABASE IF NOT EXISTS idseq_test;"
OFFLINE=1 docker compose --env-file web.env run --rm -e RAILS_ENV=development web bin/rails db:schema:load
```
(`ssl_mode: REQUIRED` in `config/database.yml` works against MySQL 5.7's self-signed cert; no SSL setup needed.)

## 3. Build the frontend bundles
The Docker image builds them, but the `.:/app` bind mount masks them at runtime, so build
once against the mounted volume:
```bash
docker compose up -d --no-deps web
docker compose exec web bash -c "npm ci --omit=optional && npx webpack --config webpack.config.dev.js"
```

## 4. (Re)start web so sprockets registers the freshly-built `app/assets/dist`
```bash
docker compose restart web
```
Use `--no-deps` so only `web` starts (skips the optional `opensearch` / `shoryuken` services,
which the app does not need to boot in `OFFLINE` mode).

## 5. Verify
```bash
curl -i http://localhost:3001/health_check     # -> HTTP 200
open http://localhost:3001/                     # full UI (Chan Zuckerberg ID)
```

## Notes
- The JWE token service reads its private key from the PEM mounted at
  `/tmp/czid-private-key.pem` (`./test/fixtures/czid-private-key.pem`); no Secrets Manager call.
- The `local-lambdas` compose profile pulls `idseq-local-*-lambda` images from ECR — **leave it
  off offline** (don't set `COMPOSE_PROFILES=local-lambdas`); the core app does not need them.
- First request to `/` is slow (~10s) while sprockets digests the bundles; cached afterward.
