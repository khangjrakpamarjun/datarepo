## docker_compose_README.md

This document explains the `docker-compose.yaml` used for the Spark + Iceberg + MinIO local development environment in this module.

## Purpose

Provide a concise explanation of each service, what ports and volumes are exposed, environment variables used, and quick-run and troubleshooting steps so you can get the environment running and understand how the pieces interact.

## Overview

The Compose file sets up a local stack for experimenting with Apache Spark and the Iceberg table format using MinIO as an S3-compatible object store. It includes:

- `spark-iceberg`: Spark environment (Jupyter + Spark UIs) with Iceberg support
- `rest`: Iceberg REST catalog service
- `minio`: MinIO S3-compatible object storage
- `mc`: MinIO client container used to initialize the `warehouse` bucket and policies

All services share a user-defined bridge network `iceberg_net` so they can address each other by service name.

## Services (summary)

### spark-iceberg
- Purpose: Main Spark environment with Jupyter notebooks and Spark UI, preconfigured to talk to Iceberg + MinIO.
- Image: `tabulario/spark-iceberg` (built from `spark/` in the repo)
- Volumes (local -> container):
  - `./warehouse` -> `/home/iceberg/warehouse` (Iceberg warehouse root)
  - `./notebooks` -> `/home/iceberg/notebooks/notebooks` (Jupyter notebooks)
  - `./data` -> `/home/iceberg/data` (sample data)
- Environment (S3/minio credentials):
  - `AWS_ACCESS_KEY_ID=admin`
  - `AWS_SECRET_ACCESS_KEY=password`
  - `AWS_REGION=us-east-1`
- Ports exposed to host:
  - `8888` -> Jupyter Notebook
  - `8080` -> Spark Master UI
  - `10000`, `10001` -> Additional Spark services (if used)
  - `4040-4042` -> Spark application UIs
- Depends on: `rest`, `minio`

Notes: The container uses the AWS-style environment variables so Spark/Iceberg can access MinIO as an S3 endpoint.

### rest (Iceberg REST catalog)
- Purpose: Iceberg REST catalog endpoint which provides a catalog API backed by S3 (MinIO).
- Image: `tabulario/iceberg-rest`
- Port: `8181` (REST API)
- Important environment variables:
  - `CATALOG_WAREHOUSE=s3://warehouse/` — Iceberg warehouse root path (S3 URL)
  - `CATALOG_IO__IMPL=org.apache.iceberg.aws.s3.S3FileIO` — Use S3 file IO implementation
  - `CATALOG_S3_ENDPOINT=http://minio:9000` — Point to MinIO service
  - Also uses the same AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY

Notes: The REST service will read and write Iceberg metadata into the configured S3-compatible endpoint (MinIO).

### minio
- Purpose: S3-compatible object storage used as the Iceberg warehouse backend.
- Image: `minio/minio`
- Ports:
  - `9000` -> MinIO API (S3-compatible access)
  - `9001` -> MinIO Console (web UI)
- Environment:
  - `MINIO_ROOT_USER=admin`
  - `MINIO_ROOT_PASSWORD=password`
- Command: `server /data --console-address :9001`
- Network alias: `warehouse.minio` (so other containers can resolve by that alias)

Notes: Default credentials are present for convenience in local dev — change them for shared or production environments.

### mc (MinIO Client container)
- Purpose: Initialize MinIO by creating the `warehouse` bucket and set a public policy. Keeps running with `tail -f /dev/null` so you can enter the container if necessary.
- Image: `minio/mc`
- Actions run at container start (entrypoint script):
  - Repeatedly attempt to configure an alias for the MinIO server: `mc alias set minio http://minio:9000/ admin password`
  - Create bucket `minio/warehouse` (no-op if it already exists)
  - Set bucket policy to `public` for ease of access to files during local development
  - `tail -f /dev/null` keeps the container alive

Notes: The `mc` container is used so you don't need to run `mc` commands manually on the host. It tolerates the bucket already existing and will leave the service running.

## Network
All services are placed in the `iceberg_net` network so they can reach one another using service names (for example, `minio:9000` from other containers).

## Where data lives
- Local repository folders are mounted into containers:
  - `./warehouse` — Iceberg metadata and table data (S3-like bucket mounted to container path)
  - `./notebooks` — Jupyter notebooks you can edit locally
  - `./data` — example CSV/inputs used in labs

Because the `warehouse` directory is mounted, created Iceberg tables and data persist on the host filesystem between container restarts.

## How to run (quick)
1. Make sure Docker is installed and running on your machine.
2. From this folder (`intermediate-bootcamp/materials/3-spark-fundamentals`), run:

```bash
# bring up everything in the background
docker compose up -d

# follow logs for a service, e.g. spark-iceberg
docker compose logs -f spark-iceberg
```

3. Access key UIs:
- Jupyter Notebook: http://localhost:8888
- Spark UI (master): http://localhost:8080
- Iceberg REST API: http://localhost:8181
- MinIO Console: http://localhost:9001 (login with `admin` / `password`)

Note: The first startup may take time as the images build/download. The `mc` container initializes the `warehouse` bucket automatically.

## Environment & credentials
The compose file uses simple credentials for convenience:
- MinIO: `MINIO_ROOT_USER=admin`, `MINIO_ROOT_PASSWORD=password`
- AWS-style env vars used by Spark/Iceberg: `AWS_ACCESS_KEY_ID=admin`, `AWS_SECRET_ACCESS_KEY=password`, `AWS_REGION=us-east-1`

Change these if you plan to expose this environment on a shared network.

## Common issues & troubleshooting
- Port conflicts: If ports like `8888`, `9000`, `9001`, or `8181` are already in use on your host, either stop the conflicting process or update the port mapping in `docker-compose.yaml`.
- Bucket already exists message: The `mc` startup script tolerates the bucket existing; you may see `Warehouse already exists` which is benign.
- If services don't connect to MinIO: ensure `minio` is healthy and reachable via `docker compose logs minio`. From another container you can test connectivity by running `/usr/bin/mc` or curling `http://minio:9000`.
- Permissions: If files written to `./warehouse` are owned by root or another user, adjust permissions on the host (e.g., `chown -R $(id -u):$(id -g) ./warehouse`).
- To see all logs:

```bash
# show logs for all services
docker compose logs -f
```

## Security note
This compose file is strictly for local development and learning: it uses cleartext credentials and a public bucket policy. Do not reuse these credentials or policies in production or on networks you don't control.

## Where to look next
- `spark/` directory (referenced by the `spark-iceberg` build) to see Dockerfile and any custom Spark / Iceberg setup
- `./notebooks` for example notebooks you can run
- `./data` for sample datasets used in labs

---

If you'd like, I can also:
- Add a short `docker compose` command snippet to a shell script to speed up startup
- Add a short checklist to the README that confirms the services are healthy after startup (commands to run and expected outputs)

