# PostgreSQL Cluster + Patroni dengan etcd dan Load Balancer

## Versi Software

* **PostgreSQL**: 16
* **Patroni**: 3.3.8
* **etcd**: 3.5.23
* **HAProxy**: Latest dari Ubuntu 22.04 repository

## Arsitektur Cluster

```
Cluster Name: pulsalink
IP Range: 10.212.100.70 - 10.212.100.73

10.212.100.70 - etcd1 + postgresql1 (primary)
10.212.100.71 - etcd2 + postgresql2 (replica)
10.212.100.72 - etcd3 + postgresql3 (replica)
10.212.100.73 - haproxy (load balancer)
```

---

## 1. Persiapan Semua Node

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y wget curl gnupg2 software-properties-common apt-transport-https
```

### Hostname

```bash
# Node 1
sudo hostnamectl set-hostname pulsalink-node1
# Node 2
sudo hostnamectl set-hostname pulsalink-node2
# Node 3
sudo hostnamectl set-hostname pulsalink-node3
```

### /etc/hosts

```bash
sudo tee -a /etc/hosts << EOF
10.212.100.70 pulsalink-node1 etcd1
10.212.100.71 pulsalink-node2 etcd2
10.212.100.72 pulsalink-node3 etcd3
10.212.100.73 pulsalink-lb haproxy
EOF
```

---

## 2. Install & Konfigurasi etcd

### Install etcd 3.5.23

```bash
ETCD_VER=v3.5.23
DOWNLOAD_URL=https://github.com/etcd-io/etcd/releases/download

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
cd /tmp && tar xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz

sudo mv etcd-${ETCD_VER}-linux-amd64/etcd /usr/local/bin/
sudo mv etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/
sudo chmod +x /usr/local/bin/etcd*

sudo useradd --system --home /var/lib/etcd --shell /bin/false etcd
sudo mkdir -p /var/lib/etcd /etc/etcd
sudo chown -R etcd:etcd /var/lib/etcd /etc/etcd
```

---

### Node1 (10.212.100.70)

```bash
# /etc/etcd/etcd.conf
sudo tee /etc/etcd/etcd.conf << 'EOF'
name: etcd1
data-dir: /var/lib/etcd
listen-peer-urls: http://10.212.100.70:2380
listen-client-urls: http://10.212.100.70:2379,http://127.0.0.1:2379
initial-advertise-peer-urls: http://10.212.100.70:2380
advertise-client-urls: http://10.212.100.70:2379
initial-cluster: etcd1=http://10.212.100.70:2380,etcd2=http://10.212.100.71:2380,etcd3=http://10.212.100.72:2380
initial-cluster-token: pulsalink-etcd-cluster
initial-cluster-state: new
heartbeat-interval: 100
election-timeout: 1000
logger: zap
log-level: info
EOF
```

### Node2 (10.212.100.71)

```bash
# /etc/etcd/etcd.conf
sudo tee /etc/etcd/etcd.conf << 'EOF'
name: etcd2
data-dir: /var/lib/etcd
listen-peer-urls: http://10.212.100.71:2380
listen-client-urls: http://10.212.100.71:2379,http://127.0.0.1:2379
initial-advertise-peer-urls: http://10.212.100.71:2380
advertise-client-urls: http://10.212.100.71:2379
initial-cluster: etcd1=http://10.212.100.70:2380,etcd2=http://10.212.100.71:2380,etcd3=http://10.212.100.72:2380
initial-cluster-token: pulsalink-etcd-cluster
initial-cluster-state: new
heartbeat-interval: 100
election-timeout: 1000
logger: zap
log-level: info
EOF
```

### Node3 (10.212.100.72)

```bash
# /etc/etcd/etcd.conf
sudo tee /etc/etcd/etcd.conf << 'EOF'
name: etcd3
data-dir: /var/lib/etcd
listen-peer-urls: http://10.212.100.72:2380
listen-client-urls: http://10.212.100.72:2379,http://127.0.0.1:2379
initial-advertise-peer-urls: http://10.212.100.72:2380
advertise-client-urls: http://10.212.100.72:2379
initial-cluster: etcd1=http://10.212.100.70:2380,etcd2=http://10.212.100.71:2380,etcd3=http://10.212.100.72:2380
initial-cluster-token: pulsalink-etcd-cluster
initial-cluster-state: new
heartbeat-interval: 100
election-timeout: 1000
logger: zap
log-level: info
EOF
```

### Systemd Service

```bash
# /etc/systemd/system/etcd.service
sudo tee /etc/systemd/system/etcd.service << 'EOF'
[Unit]
Description=etcd distributed reliable key-value store
Wants=network-online.target
After=network-online.target

[Service]
Type=notify
User=etcd
Group=etcd
WorkingDirectory=/var/lib/etcd
ExecStart=/usr/local/bin/etcd --config-file /etc/etcd/etcd.conf
Restart=on-failure
RestartSec=5
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
EOF
```

---

## 3. Install PostgreSQL 16 + Patroni 3.3.8

```bash
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/postgres.gpg

echo "deb [signed-by=/usr/share/keyrings/postgres.gpg] \
http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" | \
  sudo tee /etc/apt/sources.list.d/pgdg.list

sudo apt update
sudo apt install -y postgresql-16 postgresql-client-16 postgresql-contrib-16
sudo systemctl disable --now postgresql
```

### Install Patroni

```bash
sudo apt install -y python3-pip python3-dev libpq-dev python3-venv
sudo python3 -m venv /opt/patroni-venv
sudo /opt/patroni-venv/bin/pip install --upgrade pip
sudo /opt/patroni-venv/bin/pip install patroni[etcd3]==3.3.8 psycopg2-binary

sudo ln -sf /opt/patroni-venv/bin/patroni /usr/local/bin/patroni
sudo ln -sf /opt/patroni-venv/bin/patronictl /usr/local/bin/patronictl
```

---

## 4. Konfigurasi Patroni

### Template Patroni (Node1)
```bash
sudo mkdir -p /etc/patroni
sudo tee /etc/patroni/patroni.yml << 'EOF'
scope: pulsalink
namespace: /service/
name: postgresql1

restapi:
  listen: 10.212.100.70:8008
  connect_address: 10.212.100.70:8008

etcd3:
  hosts: 10.212.100.70:2379,10.212.100.71:2379,10.212.100.72:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    primary_start_timeout: 300
    maximum_lag_on_failover: 1048576
    postgresql:
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replicator 10.212.100.0/24 md5
    - host all all 10.212.100.0/24 md5
    - host all all 127.0.0.1/32 md5
    - host all all 10.212.100.70/32 md5

  users:
    admin:
      password: ${PATRONI_ADMIN_PASS}
      options:
        - createrole
        - createdb

postgresql:
  listen: 10.212.100.70:5432
  connect_address: 10.212.100.70:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: ${REPL_PASS}
    superuser:
      username: postgres
      password: ${POSTGRES_SUPER_PASS}
    rewind:
      username: replicator
      password: ${REPL_PASS}
  parameters:
    unix_socket_directories: '/var/run/postgresql'
EOF
```

---

### Node2 (10.212.100.71)

```bash
sudo mkdir -p /etc/patroni
sudo tee /etc/patroni/patroni.yml << 'EOF'
scope: pulsalink
namespace: /service/
name: postgresql2

restapi:
  listen: 10.212.100.71:8008
  connect_address: 10.212.100.71:8008

etcd3:
  hosts: 10.212.100.70:2379,10.212.100.71:2379,10.212.100.72:2379

postgresql:
  listen: 10.212.100.71:5432
  connect_address: 10.212.100.71:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: ${REPL_PASS}
    superuser:
      username: postgres
      password: ${POSTGRES_SUPER_PASS}
    rewind:
      username: replicator
      password: ${REPL_PASS}
  parameters:
    unix_socket_directories: '/var/run/postgresql'
EOF
```

---

### Node3 (10.212.100.72)

```bash
sudo mkdir -p /etc/patroni
sudo tee /etc/patroni/patroni.yml << 'EOF'
scope: pulsalink
namespace: /service/
name: postgresql3

restapi:
  listen: 10.212.100.72:8008
  connect_address: 10.212.100.72:8008

etcd3:
  hosts: 10.212.100.70:2379,10.212.100.71:2379,10.212.100.72:2379

postgresql:
  listen: 10.212.100.72:5432
  connect_address: 10.212.100.72:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: ${REPL_PASS}
    superuser:
      username: postgres
      password: ${POSTGRES_SUPER_PASS}
    rewind:
      username: replicator
      password: ${REPL_PASS}
  parameters:
    unix_socket_directories: '/var/run/postgresql'
EOF
```

---

## 5. Systemd Patroni

```bash
# /etc/systemd/system/patroni.service
sudo tee /etc/systemd/system/patroni.service << 'EOF'
[Unit]
Description=High availability PostgreSQL Cluster
After=network.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Setup permissions dan start Patroni
```bash
# Set ownership
sudo chown postgres:postgres /etc/patroni/patroni.yml
sudo chmod 0600 /etc/patroni/patroni.yml

export ETCDCTL_API=3
EP="--endpoints=http://10.212.100.70:2379,http://10.212.100.71:2379,http://10.212.100.72:2379"
etcdctl $EP del --prefix /service/pulsalink >/dev/null
# verifikasi: harus kosong (tidak ada output)
etcdctl $EP get --prefix /service/pulsalink


# Remove existing PostgreSQL data (akan dibuat ulang oleh Patroni)
sudo systemctl stop postgresql
sudo rm -rf /var/lib/postgresql/16/main/*

# Start Patroni
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni
sudo systemctl status patroni
```
---

## 6. HAProxy (Node 10.212.100.73)

```bash
# /etc/haproxy/haproxy.cfg
sudo tee /etc/haproxy/haproxy.cfg << 'EOF'
global
    log /dev/log local0
    user haproxy
    group haproxy
    daemon

defaults
    mode tcp
    log global
    option tcplog
    option dontlognull
    retries 3
    timeout connect 10s
    timeout client 1m
    timeout server 1m
    maxconn 300

listen stats
    bind *:7000
    mode http
    stats enable
    stats uri /
    stats realm HAProxy\ Stats
    stats admin if TRUE

# PostgreSQL Master (Write)
listen pulsalink-master
    bind *:5000
    mode tcp
    option tcp-check
    tcp-check connect port 8008
    tcp-check send GET\ /master\ HTTP/1.0\r\n\r\n
    tcp-check expect string 200
    server postgresql1 10.212.100.70:5432 check port 8008
    server postgresql2 10.212.100.71:5432 check port 8008 backup
    server postgresql3 10.212.100.72:5432 check port 8008 backup

# PostgreSQL Replica (Read)
listen pulsalink-replica
    bind *:5001
    mode tcp
    option tcp-check
    tcp-check connect port 8008
    tcp-check send GET\ /replica\ HTTP/1.0\r\n\r\n
    tcp-check expect string 200
    balance roundrobin
    server postgresql2 10.212.100.71:5432 check port 8008
    server postgresql3 10.212.100.72:5432 check port 8008
    server postgresql1 10.212.100.70:5432 check port 8008 backup
EOF
```

---

## 7. Connection String

```
# Master (Write)
postgresql://user:pass@10.212.100.73:5000/dbname

# Replica (Read)
postgresql://user:pass@10.212.100.73:5001/dbname
```

---

## 8. Cek status cluster
```bash
# Cek status etcd cluster
etcdctl --endpoints=http://10.212.100.70:2379,http://10.212.100.71:2379,http://10.212.100.72:2379 endpoint health
etcdctl --endpoints=http://10.212.100.70:2379,http://10.212.100.71:2379,http://10.212.100.72:2379 member list

# Cek status patroni cluster
sudo patronictl -c /etc/patroni/patroni.yml list

# Cek leader election
sudo patronictl -c /etc/patroni/patroni.yml show-config
```
