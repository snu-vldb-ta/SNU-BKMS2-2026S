# Oracle 26ai Free in Docker

This guide explains how to pull, run, and access Oracle AI Database Free in Docker.

## Prerequisites

- Install Docker Desktop: https://www.docker.com/products/docker-desktop/
- Oracle Container Registry: https://container-registry.oracle.com

## Check the Oracle image

Open Oracle Container Registry and navigate to:

`Database -> free -> Oracle AI Database 26ai Free Container Image Documentation`

Then check **Pull Command for Latest**.

## Pull the Oracle image

```bash
docker pull container-registry.oracle.com/database/free:latest
```

## Run the Oracle container

```bash
docker run -d --name oracle-free \
  -p 1521:1521 \
  -p 5500:5500 \
  -e ORACLE_PWD="1234" \
  container-registry.oracle.com/database/free:latest
```

## Enter the container

```bash
docker exec -it oracle-free bash
```

## Connect to Oracle using SQL*Plus

```bash
sqlplus system/1234@//localhost:1521/FREEPDB1
```

## Connection information

- **Username:** `system`
- **Password:** `1234`
- **Service name:** `FREEPDB1`

## Notes

- Replace `1234` with your own password if needed.
- The password used in `sqlplus` must match the value of `ORACLE_PWD`.

