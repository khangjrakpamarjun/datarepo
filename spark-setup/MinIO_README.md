# MinIO — quick reference for this lab

This file describes what MinIO is, why the repository uses it, how to access it (from your host and from the `mc` container included in the Compose stack), quick commands, common issues, and security notes.

## What is MinIO?
MinIO is an open-source, S3-compatible object storage server. It implements the Amazon S3 API so applications and tools that understand S3 (AWS SDKs, s3fs, Iceberg, Spark, etc.) can use a local or self-hosted object store with minimal changes.

Why use it here
- Provides a local S3-compatible backend for Iceberg and Spark to read and write table data.
- Avoids needing an AWS account during local development and labs.
- Fast and lightweight to run via Docker.

## How this repo uses MinIO
- The `docker-compose.yaml` starts a `minio` service exposing the S3 API at `http://minio:9000` inside the Docker network and at `http://localhost:9000` on the host.
- The `minio` container also exposes a management console at `http://localhost:9001`.
- Iceberg REST and Spark are configured to use MinIO as the `CATALOG_S3_ENDPOINT` or S3 endpoint, plus AWS-style credentials (set in the Compose file) so they can perform normal S3 operations.
- A small `mc` (MinIO Client) container in the stack initializes a `warehouse` bucket used as the Iceberg warehouse.

## Where to look
- Compose file: `docker-compose.yaml` in this folder — shows ports, env vars, and `mc` initialization script.
- Local data mounts:
  - `./warehouse` — mounted into containers and stores Iceberg data/metadata persistently.
  - `./notebooks` — your Jupyter notebooks.
  - `./data` — sample datasets used in labs.

## Accessing MinIO (host)
- MinIO API (S3): http://localhost:9000
- MinIO Console (web UI): http://localhost:9001
- Default (development) credentials used in this compose:
  - Access key / user: `admin`
  - Secret / password: `password`

Example: open the MinIO Console in your browser and sign in with the credentials above.

## Common host commands
Start the full stack (from this directory):

```bash
docker compose up -d
```

Start only MinIO (if you prefer to bring it up first):

```bash
docker compose up -d minio
```

Check logs for MinIO:

```bash
docker compose logs -f minio
```

Test reachability with curl (basic HTTP):

```bash
curl http://localhost:9000
```

You should get a response (typically an HTML or XML response depending on the MinIO version).

## Using the MinIO client (`mc`)
This repo already includes an `mc` service in the Compose stack which runs a small initialization script. You can also use the `mc` client from the host (install `mc`) or exec into the `mc` container.

Run a command in the running `mc` container (preferred when using the compose stack):

```bash
# run a one-off mc command inside the running mc container
docker compose exec mc sh -c "mc --help"

# list buckets on the MinIO server (mc alias set is already run by the container entrypoint)
docker compose exec mc sh -c "mc ls minio"
```

If you want to run `mc` from your host, first set the alias:

```bash
mc alias set minio http://localhost:9000 admin password
mc ls minio
mc mb minio/warehouse    # create the warehouse bucket (the mc container already does this automatically)
```

## How the `mc` container in the stack initializes MinIO
The `mc` container entrypoint in `docker-compose.yaml`:
- Repeatedly attempts to configure an alias for the MinIO server: `mc alias set minio http://minio:9000/ admin password`
- Creates bucket `minio/warehouse` if missing
- Sets a public policy on the `warehouse` bucket for easy access in local dev
- Leaves the container running (`tail -f /dev/null`) so you can `exec` into it for manual commands

This means on a fresh start the bucket should be available without manual steps.

## Troubleshooting
- Ports in use: If `9000` or `9001` are in use on your host, change the host port mappings in `docker-compose.yaml` or stop the conflicting process.
- "Bucket already exists": benign — the initialization script tolerates the bucket existing.
- Can't reach MinIO from other containers: ensure services are on the same Docker network (`iceberg_net` in the Compose file). From inside another container, try:

```bash
# example run inside spark-iceberg container
docker compose exec spark-iceberg sh -c "curl -I http://minio:9000"
```

- Permissions on `./warehouse`: If files are owned by root, change them on the host so local edits are easier:

```bash
sudo chown -R $(id -u):$(id -g) ./warehouse
```

## Security notes
- The Compose stack uses cleartext, convenient credentials (`admin`/`password`) and a public bucket policy for local development only. Do NOT reuse these credentials in production.
- For production or shared environments, enable TLS, stronger credentials, and proper network isolation. Consider MinIO's distributed mode for production-scale deployments.

## Example: connect Spark/Iceberg to MinIO (concept)
- Spark/Iceberg use the AWS-style environment variables in the Compose file
  - `AWS_ACCESS_KEY_ID=admin`
  - `AWS_SECRET_ACCESS_KEY=password`
  - `CATALOG_S3_ENDPOINT=http://minio:9000` (set in the Iceberg REST service)
- When Spark/Iceberg use an S3FileIO implementation and point to the MinIO endpoint, they can read/write Iceberg tables just like they would with real S3.
