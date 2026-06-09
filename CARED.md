# Cared Appwrite Cloud Adaptation

This repository carries a small Cared-specific adaptation for running Appwrite Cloud in Cared's self-hosted environment.

## Upstream Tracking

- Always track the latest upstream cloud tag in the `cl-1.9.0-x` series.
- Create the Cared branch from the exact upstream tag being adapted.
- Branch names include the suffix: `cl-1.9.0-x-cared`.
- Keep the Cared adaptation as a small, reviewable commit on top of the upstream tag whenever practical.

Example: `cl-1.9.0-5-cared` was created from upstream tag `cl-1.9.0-5` and contains the Cared-specific adaptation on top.

## Image Build Rule

The Docker build `VERSION` argument and image tag must use the upstream tag name without the `-cared` suffix.

Example:

```sh
docker buildx build --build-arg VERSION="cl-1.9.0-5" --platform linux/amd64 -t caredai/appwrite:cl-1.9.0-5 .
```

## Adaptation Goals

The goal is to make upstream Appwrite Cloud work in Cared's self-hosted deployment without depending on Appwrite's full official cloud infrastructure.

When moving to a newer `cl-1.9.0-x` tag, inspect the upstream versions of the affected files and re-apply the behavior below. Preserve the intent rather than blindly copying old hunks.

## Required Cared Changes

- In `.env`, default the database settings to PostgreSQL instead of MongoDB:
  - `_APP_DB_ADAPTER=postgresql`
  - `_APP_DB_HOST=postgresql`
  - `_APP_DB_PORT=5432`
- In `.env`, set `_APP_DATABASE_SHARED_TABLES=database_db_main` so the deployment uses the expected shared database table configuration.
- In `.gitignore`, ignore local Appwrite component checkouts:
  - `console/`
  - `proxy/`
  - `executor/`
  - `open-runtimes/`
  - `open-runtimes-k8s-executor/`
  - `orchestration/`
- In `app/controllers/general.php`, tolerate function execution responses with missing `headers` by iterating over `($executionResponse['headers'] ?? [])`.
- In `app/init/registers.php`, make the Redis queue broker connection preserve DSN credentials and authenticate with the password before returning the Redis client. Cared's Redis deployment may provide credentials in the broker DSN.
- In `src/Appwrite/Certificates/LetsEncrypt.php`, prevent Appwrite from writing generated Traefik dynamic config files after certificate issuance. Cared manages routing/config outside this Appwrite container, while Appwrite still needs to read and parse the issued certificate.
- In `src/Appwrite/Platform/Modules/Project/Http/Project/Keys/Create.php`, allow `POST /v1/project/keys` to accept an optional `secret` parameter. Validate that it matches `API_KEY_STANDARD . '_<256 hex characters>'`; use it when provided, otherwise keep upstream random secret generation. This lets Cared provision deterministic/pre-generated API keys during integration.
