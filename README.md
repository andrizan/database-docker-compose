# 🗄️ Database & Service Stack Collection

A collection of configuration templates, Docker Compose setups, and deployment examples for commonly used databases and services in internal development environments.

---

## 📂 Directory Structure

| Folder           | Description                                                       |
| ---------------- | ----------------------------------------------------------------- |
| `elastic/`       | Elasticsearch configuration for full-text search.                 |
| `gitlab-runner/` | GitLab Runner configuration (cache, autoscaling, executor, etc.). |
| `meilisearch/`   | Meilisearch setup for lightweight, fast text search.              |
| `minio/`         | MinIO setup for S3-compatible object storage.                     |
| `mongodb/`       | MongoDB configuration.                                            |
| `mysql/`         | MySQL configuration and version updates.                          |
| `pgcluster-zoo/` | PostgreSQL cluster setup with Patroni, HAProxy, and ETCD.         |
| `postgresql/`    | Standalone PostgreSQL instance with Grafana JSON dashboards.      |
| `rabbitmq/`      | RabbitMQ message broker setup.                                    |
| `redis/`         | Redis/Valkey instance for caching and message queues.             |
| `scylla-db/`     | ScyllaDB (Cassandra-compatible) setup.                            |
| `sonarqube-ce/`  | SonarQube Community Edition setup for static code analysis.       |
| `sqlserver/`     | Microsoft SQL Server (MSSQL) configuration with ODBC integration. |

---

## ⚙️ System Requirements

* **Docker Engine** ≥ 24.x
* **Docker Compose** ≥ 2.x
* Minimum **4 vCPUs** and **8 GB RAM** recommended
* For PostgreSQL clusters: additional nodes required for Patroni, HAProxy, and ETCD

---

## 🚀 Usage

1. Choose the desired service folder (e.g. `postgresql/`).
2. Edit the `.env` file if available to adjust configuration values.
3. Start the service:

   ```bash
   docker compose up -d
   ```
4. Access the service via the port defined in `docker-compose.yml`.

---

## 📊 Monitoring & Observability

Some services (such as PostgreSQL and Redis) include **Prometheus** and **Grafana** integrations.
Example Grafana dashboards are provided under the `postgresql/` directory.

---

## 🔐 Security Notes

* Store credentials and secrets in `.env` files (never commit them).
* For production deployments, always use a TLS proxy (e.g. Nginx + Let’s Encrypt).
* Apply strong passwords and network-level access control for exposed ports.

---

## 🧾 License

[Unlicense](/LICENSE)
