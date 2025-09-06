# FleetDM on Docker — Stack README

This stack runs FleetDM with MySQL and Redis in Docker. All services are internal-only; external access is provided through a reverse proxy (e.g., Nginx Proxy Manager) on a shared `proxy` network.

---

## What this stack provides

* **FleetDM** app server (UI/API + osquery enroll endpoint)
* **MySQL** as the primary data store
* **Redis** for background jobs and caching
* **Health checks** to ensure dependency ordering
* **Named volumes** for persistence
* **No host ports published** — only exposed to other containers on the `proxy` network

---

## Architecture

```
[Internet]
    |
  HTTPS
    |
[Nginx Proxy Manager]  (on external 'proxy' network)
    |            \
    |             \  TCP 8220 (stream)
HTTP 8080          \
    |               \
[fleet:8080]     [fleet:8220]
    |
  depends_on
    |
[redis:6379]   [mysql:3306]
```

* TLS is terminated by NPM.
* Fleet listens on plain HTTP (`8080`) inside the cluster.
* The osquery enroll endpoint uses raw TCP on port `8220`.

---

## Dependencies and assumptions

* **Docker Engine v24+** and **Docker Compose v2** on a 64-bit Linux host.
* An external Docker network named **`proxy`** that your reverse proxy container also uses.

  ```bash
  docker network create proxy
  ```
* Nginx Proxy Manager (NPM) or equivalent, attached to the `proxy` network.
* DNS pointing `fleet.example.com` at your reverse proxy host.
* TLS handled entirely by the proxy.

---

## Environment variables

Create a `.env` file alongside the compose file:

```env
# Timezone
TZ=America/New_York

# MySQL credentials
MYSQL_ROOT_PASSWORD=change_me_root
MYSQL_DATABASE=fleet
MYSQL_USER=fleet
MYSQL_PASSWORD=change_me_user

# Fleet configuration
FLEET_LOGGING_JSON=true
FLEET_OSQUERY_STATUS_LOG_PLUGIN=filesystem
FLEET_FILESYSTEM_STATUS_LOG_FILE=/logs/osquery_status.log
FLEET_FILESYSTEM_RESULT_LOG_FILE=/logs/osquery_result.log
FLEET_LICENSE_KEY=

# Vulnerability settings
FLEET_OSQUERY_LABEL_UPDATE_INTERVAL=1h
FLEET_VULNERABILITIES_CURRENT_INSTANCE_CHECKS=true
FLEET_VULNERABILITIES_DATABASES_PATH=/vulndb
FLEET_VULNERABILITIES_PERIODICITY=1h

# Only needed if forcing container user
PUID=1000
PGID=1000
```

---

## Reverse proxy configuration (NPM)

In **Nginx Proxy Manager**:

1. **HTTP Host Proxy**

   * **Domain**: `fleet.example.com`
   * **Forward Hostname / IP**: `fleet`
   * **Forward Port**: `8080`
   * Enable SSL and request a Let’s Encrypt certificate.
   * Force SSL.

2. **TCP Stream Proxy**

   * Add a new **Stream** in NPM.
   * **Listen Port**: `8220`
   * **Forward Hostname / IP**: `fleet`
   * **Forward Port**: `8220`
   * This is a raw TCP proxy. Do not wrap it in HTTP.

---

## Running the stack

Bring up the services:

```bash
docker compose --env-file .env up -d
```

* Fleet will run `prepare db` automatically on startup.
* Health checks ensure MySQL and Redis are ready before Fleet starts.

Access Fleet at:

```
https://fleet.example.com
```

---

## Volumes

* `mysql` — MySQL data
* `redis` — Redis AOF data
* `data` — Fleet application state
* `logs` — Local Fleet logs (if using filesystem log plugin)
* `vulndb` — Cached vulnerability databases

Back these up regularly.

---

## Backups

1. **What to back up**:

   * `mysql` (mandatory)
   * `data`, `logs`, `vulndb` (recommended)
   * `redis` (optional; cold cache can be rebuilt)

2. **Snapshot example**:

   ```bash
   docker run --rm -v mysql:/vol -v $PWD:/backup alpine \
     tar -C /vol -czf /backup/mysql.tgz .
   ```

3. **Restore**:

   * Create empty volumes.
   * Extract backup into volumes.
   * Restart the stack.

---

## Security notes

* Do not publish MySQL or Redis ports to the host.
* Use strong, unique passwords for MySQL.
* Restrict which containers can join the `proxy` network.
* Rotate Fleet API tokens and enroll secrets regularly.

---

## Troubleshooting

* **Fleet unhealthy**:
  Check logs with `docker logs fleet` or `wget -qO- http://127.0.0.1:8080/healthz` inside the container.
* **Proxy cannot reach Fleet**:
  Confirm both NPM and Fleet are attached to the `proxy` network. From NPM container:
  `curl http://fleet:8080/healthz`
* **Agents not enrolling**:
  Verify TCP stream proxy on port `8220` is reachable externally.

---

## Scaling

* For larger installs, pin specific image tags (`mysql:8.x`, `redis:7.x`, `fleetdm/fleet:<version>`).
* Tune MySQL (`innodb_buffer_pool_size`, `utf8mb4`).
* Use external logging instead of filesystem logs.
* Scale Fleet horizontally by running multiple replicas behind the same proxy, backed by a shared MySQL and Redis.

---
