# Spark + Iceberg — quick reference for this lab

This README explains what Apache Iceberg is, how Spark integrates with it, and how the Compose stack in this repo wires Spark, the Iceberg REST catalog, and MinIO together. It includes a minimal Spark example, common operations, tips, and pointers to relevant files in this repo.

## What is Apache Iceberg?
Apache Iceberg is an open table format for large analytical datasets (Parquet/ORC/Avro files) that provides:
- ACID semantics and snapshot isolation on object stores (S3/MinIO/HDFS)
- Time travel (query previous snapshots)
- Safe schema evolution (add/rename/drop columns)
- Hidden partitioning and efficient metadata pruning for faster reads
- Manifest-based metadata to scale to large tables

Iceberg sits between compute (Spark, Flink, Trino) and object storage and manages table metadata and snapshots so reads and writes are consistent and manageable.

## How Spark and Iceberg work together
Spark can natively read and write Iceberg tables either via SQL or the DataFrame API. Spark needs the Iceberg runtime on the classpath and a configured Iceberg catalog to locate table metadata. Typical patterns:
- Use Spark SQL: CREATE TABLE ... USING iceberg
- Use DataFrame API: df.write.format("iceberg").save("catalog.db.table")

Catalogs map table identifiers to metadata storage. Examples: HadoopCatalog, HiveCatalog, RESTCatalog, GlueCatalog. This repo uses the REST catalog (an `iceberg-rest` service) with MinIO as the S3-compatible storage backend.

## Repo-specific architecture (Compose stack)
- `minio` — S3-compatible object store (s3://warehouse/) where Iceberg stores table data and metadata
- `rest` — Iceberg REST catalog service that exposes a catalog API; it reads/writes metadata to MinIO
- `spark-iceberg` — Spark environment (Jupyter + Spark) with Iceberg libraries included; configured to use the REST catalog and MinIO

Spark is configured (via environment variables / Spark configs) to use AWS-style credentials and the REST catalog URI (e.g., `http://rest:8181`). Iceberg metadata files and data files land in the `./warehouse` host folder mounted into containers.
## Tips, pitfalls & troubleshooting
- Version compatibility: ensure Spark, Iceberg library, and the REST catalog are compatible. Mismatched versions are a common source of errors.
- Credentials & endpoint: Spark and the REST catalog must have correct S3 endpoint and AWS credentials (in this repo the environment variables point to MinIO and `admin`/`password`).
- Metadata growth: Iceberg's metadata grows with snapshots. Run snapshot expiration and metadata cleanup periodically during experiments.
- Small files: streaming or frequent small writes can create many small data files; use compaction/optimize patterns.
- Local persistence: the `./warehouse` folder is mounted and persists data on the host. If ownership or permissions issues arise, run `sudo chown -R $(id -u):$(id -g) ./warehouse`.

## Where to look in this repo
- `docker-compose.yaml` — shows the services, env vars, and how Spark, rest, and MinIO are connected
- `spark/` — Docker build context for the `spark-iceberg` image (check Dockerfile to see the Iceberg jars and Spark config)
- `notebooks/` — example notebooks you can run in the Jupyter environment
- `warehouse/` — local mount that stores Iceberg data/metadata

