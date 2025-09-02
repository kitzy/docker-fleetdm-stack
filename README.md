# docker-fleetdm-stack

Docker Compose stack for running [FleetDM](https://fleetdm.com), an open-source osquery manager. The stack bundles the Fleet server with its MySQL and Redis dependencies.

## Services

| Service      | Purpose                     | Ports (host:container) |
|--------------|-----------------------------|------------------------|
| mysql-fleet  | MySQL database for Fleet    | exposes `3306`         |
| redis-fleet  | Redis cache for Fleet       | exposes `6379`         |
| fleet        | FleetDM server              | `8082:8080`, `8220:8220` |

## Prerequisites

- Docker and Docker Compose or a Portainer installation
- Host directories available for the volume paths defined in `docker-compose.yml`

## Configuration

Create a `.env` file in the project root containing the variables below. Adjust the example values for your environment.

| Variable | Example | Description |
|----------|---------|-------------|
| `TZ` | `UTC` | Time zone applied to all containers. |
| `MYSQL_ROOT_PASSWORD` | `supersecret` | Root password for MySQL. |
| `MYSQL_DATABASE` | `fleet` | Name of the Fleet database. |
| `MYSQL_USER` | `fleet` | MySQL user Fleet uses. |
| `MYSQL_PASSWORD` | `fleetpass` | Password for `MYSQL_USER`. |
| `PUID` | `1000` | UID the Fleet container runs as (controls file ownership). |
| `PGID` | `1000` | GID the Fleet container runs as. |
| `FLEET_MYSQL_ADDRESS` | `mysql-fleet:3306` | Host and port of the MySQL service. |
| `FLEET_MYSQL_DATABASE` | `fleet` | Database name used by Fleet. |
| `FLEET_MYSQL_USERNAME` | `fleet` | MySQL username for Fleet. |
| `FLEET_MYSQL_PASSWORD` | `fleetpass` | Password for Fleet's MySQL user. |
| `FLEET_REDIS_ADDRESS` | `redis-fleet:6379` | Host and port of the Redis service. |
| `FLEET_SERVER_ADDRESS` | `0.0.0.0:8080` | Address Fleet listens on inside the container. |
| `FLEET_SERVER_TLS` | `false` | Set to `true` to enable TLS for the Fleet server. |
| `FLEET_SERVER_PRIVATE_KEY` | `/fleet/server.key` | Path to the TLS private key when TLS is enabled. |
| `FLEET_LOGGING_JSON` | `true` | Emit logs in JSON format. |
| `FLEET_OSQUERY_STATUS_LOG_PLUGIN` | `filesystem` | Backend used for osquery status logs. |
| `FLEET_FILESYSTEM_STATUS_LOG_FILE` | `/logs/status.log` | Location of osquery status logs. |
| `FLEET_FILESYSTEM_RESULT_LOG_FILE` | `/logs/result.log` | Location of osquery result logs. |
| `FLEET_LICENSE_KEY` | *(empty)* | Fleet license key (leave empty for OSS). |
| `FLEET_OSQUERY_LABEL_UPDATE_INTERVAL` | `1h` | How often Fleet refreshes labels. |
| `FLEET_VULNERABILITIES_CURRENT_INSTANCE_CHECKS` | `true` | Restrict vulnerability checks to this instance. |
| `FLEET_VULNERABILITIES_DATABASES_PATH` | `/vulndb` | Directory storing vulnerability databases. |
| `FLEET_VULNERABILITIES_PERIODICITY` | `24h` | Interval between vulnerability database updates. |

## Usage

Start the stack locally:

```bash
docker compose up -d
```

Then visit `http://localhost:8082` for the Fleet UI and use port `8220` for osquery agent enrollment.

## Deploying with Portainer

1. In Portainer, navigate to **Stacks → Add stack**.
2. Choose **Git repository**, enter this repository URL, and set the compose file path to `docker-compose.yml`.
3. Supply the environment variables above via the web form or attach a `.env` file.
4. Enable **Automatic updates** to keep the stack in sync with the repository (GitOps). Select a polling interval or webhook trigger.
5. Click **Deploy the stack**.

Portainer will pull new commits and redeploy automatically when automatic updates are enabled.

## Customizing for your environment

- **Volume paths** – Adjust the host paths under `volumes` if `/mnt/data/docker/volumes/` does not exist on your system.
- **Ports** – Change the left side of port mappings such as `8082:8080` if those host ports are already in use.
- **User and group IDs** – Set `PUID` and `PGID` to match the user that should own mounted files.
- **Platform** – Modify or remove `platform: linux/x86_64` if you are running on a different CPU architecture.
- **TLS settings** – When `FLEET_SERVER_TLS=true`, mount certificate files and set `FLEET_SERVER_PRIVATE_KEY` accordingly.

## Data persistence

MySQL, Redis, and Fleet data are stored in the directories mounted under `/mnt/data/docker/volumes/`. Back up these paths to preserve data between deployments.

## Validation

Validate the compose file before deploying:

```bash
docker compose -f docker-compose.yml config
```

