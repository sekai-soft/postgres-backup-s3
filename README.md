# Introduction
This project provides Docker images to periodically back up a PostgreSQL database to AWS S3, and to restore from the backup as needed.

# Usage
## Backup
```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password

  backup:
    image: ghcr.io/k-t-corp/postgres-backup-s3:15
#    volumes:
#      - ./post.sh:/post.sh  # optionally mount a shell script at /post.sh to be executed after backup is complete, e.g. sending a heartbeat
    environment:
      SCHEDULE: '@weekly'     # optional
      BACKUP_KEEP_DAYS: 7     # optional
      PASSPHRASE: passphrase  # optional
      S3_REGION: region
      S3_ACCESS_KEY_ID: key
      S3_SECRET_ACCESS_KEY: secret
      S3_BUCKET: my-bucket
      S3_PREFIX: backup
      POSTGRES_HOST: postgres
      POSTGRES_DATABASE: dbname
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
```

- Images are tagged by the major PostgreSQL version supported: `12`, `13`, `14`, `15` or `16`.
- The `SCHEDULE` variable determines backup frequency. See go-cron schedules documentation [here](http://godoc.org/github.com/robfig/cron#hdr-Predefined_schedules). Omit to run the backup immediately and then exit.
- If `PASSPHRASE` is provided, the backup will be encrypted using GPG.
- Run `docker exec <container name> sh backup.sh` to trigger a backup ad-hoc.
- If `BACKUP_KEEP_DAYS` is set, backups older than this many days will be deleted from S3.
- Set `S3_ENDPOINT` if you're using a non-AWS S3-compatible storage provider.

## Restore
> **WARNING:** DATA LOSS! All database objects will be dropped and re-created.
### ... from latest backup
```sh
docker exec <container name> sh restore.sh
```
> **NOTE:** If your bucket has more than a 1000 files, the latest may not be restored -- only one S3 `ls` command is used
### ... from specific backup
```sh
docker exec <container name> sh restore.sh <timestamp>
```

# Development
## Build the image locally
`ALPINE_VERSION` determines Postgres version compatibility. See [`build-and-push-images.yml`](.github/workflows/build-and-push-images.yml) for the latest mapping.
```sh
DOCKER_BUILDKIT=1 docker build --build-arg ALPINE_VERSION=3.14 .
```
## Run a simple test environment with Docker Compose
```sh
cp template.env .env
# fill out your secrets/params in .env
docker compose up -d
```

# Acknowledgements
This project is a fork and re-structuring of @schickling's [postgres-backup-s3](https://github.com/schickling/dockerfiles/tree/master/postgres-backup-s3) and [postgres-restore-s3](https://github.com/schickling/dockerfiles/tree/master/postgres-restore-s3).

## Fork goals
These changes would have been difficult or impossible merge into @schickling's repo or similarly-structured forks.
  - dedicated repository
  - automated builds
  - support multiple PostgreSQL versions
  - backup and restore with one image

## Other changes and features
  - some environment variables renamed or removed
  - uses `pg_dump`'s `custom` format (see [docs](https://www.postgresql.org/docs/10/app-pgdump.html))
  - drop and re-create all database objects on restore
  - backup blobs and all schemas by default
  - no Python 2 dependencies
  - filter backups on S3 by database name
  - support encrypted (password-protected) backups
  - support for restoring from a specific backup by timestamp
  - support for auto-removal of old backups
