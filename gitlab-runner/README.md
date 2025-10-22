# Dokumentasi GitLab Runner dengan Docker Compose

## Daftar Isi
- [Ringkasan Konfigurasi](#ringkasan-konfigurasi)
- [Struktur Direktori](#struktur-direktori)
- [Prasyarat](#prasyarat)
- [Instalasi](#instalasi)
- [Konfigurasi](#konfigurasi)
- [Registrasi Runner](#registrasi-runner)
- [Penggunaan Tags](#penggunaan-tags)
- [Monitoring dan Maintenance](#monitoring-dan-maintenance)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

---

## Ringkasan Konfigurasi

### Spesifikasi Runner
- **Concurrent Jobs:** 8 job bersamaan
- **Default Image:** Ubuntu 24.04
- **Resource per Job:** 1 CPU, 2GB RAM
- **Total Resource:** 7.5 CPU, 15GB RAM
- **Cache:** Local shared cache
- **Session Timeout:** 4000 detik (66 menit)

### Tags Tersedia
- `build-autoscale` - Untuk job build
- `test-autoscale` - Untuk job testing
- `release-autoscale` - Untuk job release
- `shared-autoscale` - Untuk job shared
- `deploy-autoscale` - Untuk job deployment

---

## Struktur Direktori

```
gitlab-runner/
├── config.toml              # Konfigurasi GitLab Runner
├── docker-compose.yaml      # Konfigurasi Docker Compose
└── cache/                   # Direktori cache (dibuat otomatis)
```

### File: `docker-compose.yaml`

```yaml
services:
  gitlab-runner:
    image: gitlab/gitlab-runner:ubuntu-v18.5.0
    container_name: gitlab-runner
    restart: always
    volumes:
      - ./config.toml:/etc/gitlab-runner/config.toml
      - /var/run/docker.sock:/var/run/docker.sock
      - ./cache:/cache
    environment:
      - DOCKER_DRIVER=overlay2
    deploy:
      resources:
        limits:
          cpus: '7.5'
          memory: 15G
        reservations:
          cpus: '4'
          memory: 8G
    networks:
      - gitlab-runner-network

networks:
  gitlab-runner-network:
    driver: bridge
```

### File: `config.toml`

```toml
concurrent = 8
check_interval = 0

[session_server]
  session_timeout = 4000

[[runners]]
  name = "docker-runner-01"
  url = "https://gitlab.com/"
  token = "YOUR_RUNNER_TOKEN"
  executor = "docker"
  tag_list = ["build-autoscale", "test-autoscale", "release-autoscale", "shared-autoscale", "deploy-autoscale"]

  [runners.custom_build_dir]

  [runners.cache]
    Type = "local"
    Path = "/cache"
    Shared = true

    [runners.cache.local]

  [runners.docker]
    tls_verify = false
    image = "ubuntu:24.04"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 2147483648

    cpus = "1"
    memory = "2g"
    memory_swap = "2g"
    memory_reservation = "1g"

    network_mode = "bridge"
    pull_policy = ["if-not-present"]
    helper_image = "gitlab/gitlab-runner-helper:latest"

  [runners.docker.tmpfs]
    "/tmp" = "rw,noexec,nosuid,size=512m"
    "/run" = "rw,noexec,nosuid,size=512m"
```

---

## Prasyarat

### Spesifikasi Server
- **CPU:** 8 Core
- **RAM:** 16 GB
- **Storage:** 100 GB SSD
- **OS:** Ubuntu 20.04/22.04/24.04 atau CentOS 7/8

### Software yang Dibutuhkan
- Docker Engine 20.10+
- Docker Compose 2.0+

### Instalasi Docker dan Docker Compose

```bash
# Update sistem
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo apt install docker-compose-plugin -y

# Verifikasi instalasi
docker --version
docker compose version

# Tambahkan user ke grup docker
sudo usermod -aG docker $USER
newgrp docker
```

---

## Instalasi

### Langkah 1: Buat Struktur Direktori

```bash
# Buat direktori utama
mkdir -p ~/gitlab-runner/cache
cd ~/gitlab-runner
```

### Langkah 2: Buat File docker-compose.yaml

```bash
nano docker-compose.yaml
```

Salin isi dari section [Struktur Direktori](#struktur-direktori) di atas.

### Langkah 3: Buat File config.toml

```bash
nano config.toml
```

Salin isi dari section [Struktur Direktori](#struktur-direktori) di atas.

### Langkah 4: Konfigurasi Token

Anda perlu mengganti `YOUR_RUNNER_TOKEN` dengan token asli dari GitLab.

#### Cara Mendapatkan Token:

**Untuk Project Runner:**
1. Buka project GitLab Anda
2. Navigate ke **Settings** > **CI/CD**
3. Expand **Runners**
4. Klik **New project runner**
5. Pilih **Linux** sebagai operating system
6. Tambahkan tags jika diperlukan
7. Klik **Create runner**
8. Salin **runner authentication token** yang ditampilkan

**Untuk Instance Runner (Admin only):**
1. Login sebagai administrator
2. Navigate ke **Admin Area** > **CI/CD** > **Runners**
3. Klik **New instance runner**
4. Ikuti langkah yang sama seperti project runner

#### Edit config.toml:

```bash
nano config.toml
```

Ganti baris ini:
```toml
token = "YOUR_RUNNER_TOKEN"
```

Menjadi:
```toml
token = "glrt-xxxxxxxxxxxxxxxxxxxx"  # Token asli Anda
```

Jika menggunakan GitLab self-hosted, ganti juga URL:
```toml
url = "https://gitlab.example.com/"  # URL GitLab Anda
```

### Langkah 5: Jalankan Runner

```bash
# Jalankan container
docker compose up -d

# Verifikasi container berjalan
docker compose ps
```

Output yang diharapkan:
```
NAME              IMAGE                                    STATUS
gitlab-runner     gitlab/gitlab-runner:ubuntu-v18.5.0     Up 10 seconds
```

### Langkah 6: Verifikasi Runner

```bash
# Cek logs
docker compose logs -f gitlab-runner

# Verifikasi runner terdaftar
docker compose exec gitlab-runner gitlab-runner verify
```

Output yang diharapkan:
```
Runtime platform                                    arch=amd64 os=linux
Running in system-mode.

Verifying runner... is alive                        runner=xxxxxxxx
```

### Langkah 7: Cek di GitLab UI

1. Buka GitLab project/instance Anda
2. Navigate ke **Settings** > **CI/CD** > **Runners**
3. Runner Anda seharusnya muncul dengan status **online** (hijau)
4. Tags seharusnya terlihat: `build-autoscale`, `test-autoscale`, dll.

---

## Konfigurasi

### Parameter Konfigurasi Penting

#### config.toml

| Parameter | Nilai | Keterangan |
|-----------|-------|------------|
| `concurrent` | 8 | Jumlah job bersamaan |
| `check_interval` | 0 | Interval pengecekan job baru (0 = default) |
| `session_timeout` | 4000 | Timeout session dalam detik (66 menit) |
| `executor` | "docker" | Tipe executor yang digunakan |
| `image` | "ubuntu:24.04" | Default Docker image untuk job |
| `cpus` | "1" | Jumlah CPU per job |
| `memory` | "2g" | RAM per job |
| `shm_size` | 2147483648 | Shared memory (2GB) |

#### docker-compose.yaml

| Parameter | Nilai | Keterangan |
|-----------|-------|------------|
| `cpus (limits)` | '7.5' | Maximum CPU untuk runner |
| `memory (limits)` | 15G | Maximum RAM untuk runner |
| `cpus (reservations)` | '4' | Minimum CPU yang direservasi |
| `memory (reservations)` | 8G | Minimum RAM yang direservasi |

### Menyesuaikan Resource

Jika ingin mengubah alokasi resource, edit file terkait:

**Untuk mengubah concurrent jobs:**
```bash
nano config.toml
# Ubah: concurrent = 10
docker compose restart
```

**Untuk mengubah resource per job:**
```bash
nano config.toml
# Ubah:
# cpus = "2"
# memory = "4g"
docker compose restart
```

**Untuk mengubah total resource runner:**
```bash
nano docker-compose.yaml
# Ubah di section deploy.resources
docker compose up -d
```

---

## Registrasi Runner

Runner sudah otomatis terdaftar saat container pertama kali dijalankan jika token sudah dikonfigurasi dengan benar di `config.toml`.

### Metode Alternatif: Registrasi Manual

Jika Anda ingin menambahkan runner baru atau re-register:

#### Registrasi Interaktif:

```bash
docker compose exec gitlab-runner gitlab-runner register
```

Ikuti prompt:
- **GitLab instance URL:** `https://gitlab.com/`
- **Registration token:** Token dari GitLab
- **Description:** `docker-runner-02`
- **Tags:** `build-autoscale,test-autoscale`
- **Executor:** `docker`
- **Default Docker image:** `ubuntu:24.04`

#### Registrasi Non-Interaktif:

```bash
docker compose exec gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.com/" \
  --token "glrt-xxxxxxxxxxxxxxxxxxxx" \
  --executor "docker" \
  --docker-image "ubuntu:24.04" \
  --description "Production Docker Runner" \
  --tag-list "build-autoscale,test-autoscale,release-autoscale,shared-autoscale,deploy-autoscale" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
  --docker-volumes "/cache" \
  --docker-privileged=false \
  --docker-cpus="1" \
  --docker-memory="2g"
```

### Unregister Runner

```bash
# Unregister runner
docker compose exec gitlab-runner gitlab-runner unregister --name docker-runner-01

# Unregister semua runner
docker compose exec gitlab-runner gitlab-runner unregister --all-runners
```

---

## Penggunaan Tags

Tags digunakan untuk mengarahkan job ke runner yang sesuai. Konfigurasi ini memiliki 5 tags:

### Tags yang Tersedia

| Tag | Penggunaan | Contoh Job |
|-----|------------|------------|
| `install-autoscale` | Job install | Install deps/package |
| `build-autoscale` | Job build/compile | Compile code, build Docker image |
| `test-autoscale` | Job testing | Unit test, integration test |
| `release-autoscale` | Job release | Create release, tag version |
| `shared-autoscale` | Job general purpose | Linting, formatting, analysis |
| `deploy-autoscale` | Job deployment | Deploy ke production/staging |

### Contoh .gitlab-ci.yml

#### Contoh 1: Pipeline Sederhana

```yaml
stages:
  - build
  - test
  - deploy

build-job:
  stage: build
  image: ubuntu:24.04
  tags:
    - build-autoscale
  script:
    - apt-get update -qq
    - apt-get install -y build-essential
    - echo "Building application..."
    - gcc --version

test-job:
  stage: test
  image: ubuntu:24.04
  tags:
    - test-autoscale
  script:
    - apt-get update -qq
    - apt-get install -y python3 python3-pip
    - pip3 install pytest
    - echo "Running tests..."
    - python3 --version

deploy-production:
  stage: deploy
  image: ubuntu:24.04
  tags:
    - deploy-autoscale
  script:
    - echo "Deploying to production..."
  only:
    - main
  when: manual
```

#### Contoh 2: Pipeline dengan Multiple Tags

```yaml
stages:
  - lint
  - build
  - test
  - release
  - deploy

lint-code:
  stage: lint
  image: python:3.11
  tags:
    - shared-autoscale
  script:
    - pip install flake8
    - flake8 .

build-docker:
  stage: build
  image: docker:24.0.5
  tags:
    - build-autoscale
  services:
    - docker:24.0.5-dind
  script:
    - docker build -t myapp:latest .

unit-test:
  stage: test
  image: python:3.11
  tags:
    - test-autoscale
  script:
    - pip install -r requirements.txt
    - pytest tests/unit/

integration-test:
  stage: test
  image: python:3.11
  tags:
    - test-autoscale
  script:
    - pip install -r requirements.txt
    - pytest tests/integration/

create-release:
  stage: release
  image: ubuntu:24.04
  tags:
    - release-autoscale
  script:
    - echo "Creating release v1.0.0"
  only:
    - tags

deploy-staging:
  stage: deploy
  image: ubuntu:24.04
  tags:
    - deploy-autoscale
  script:
    - echo "Deploying to staging"
  only:
    - develop

deploy-production:
  stage: deploy
  image: ubuntu:24.04
  tags:
    - deploy-autoscale
  script:
    - echo "Deploying to production"
  only:
    - main
  when: manual
```

#### Contoh 3: Pipeline Node.js

```yaml
stages:
  - install
  - lint
  - test
  - build
  - deploy

variables:
  NODE_VERSION: "20"

install-dependencies:
  stage: install
  image: node:${NODE_VERSION}
  tags:
    - shared-autoscale
  script:
    - npm ci
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
  artifacts:
    paths:
      - node_modules/
    expire_in: 1 hour

lint:
  stage: lint
  image: node:${NODE_VERSION}
  tags:
    - shared-autoscale
  dependencies:
    - install-dependencies
  script:
    - npm run lint

test:
  stage: test
  image: node:${NODE_VERSION}
  tags:
    - test-autoscale
  dependencies:
    - install-dependencies
  script:
    - npm run test
  coverage: '/Lines\s*:\s*(\d+\.\d+)%/'

build:
  stage: build
  image: node:${NODE_VERSION}
  tags:
    - build-autoscale
  dependencies:
    - install-dependencies
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week

deploy:
  stage: deploy
  image: ubuntu:24.04
  tags:
    - deploy-autoscale
  dependencies:
    - build
  script:
    - echo "Deploying application..."
  only:
    - main
```

### Tips Penggunaan Tags

1. **Gunakan tag yang spesifik** - Pilih tag yang sesuai dengan jenis job
2. **Kombinasi tags** - Bisa menggunakan multiple tags untuk filtering lebih detail
3. **Konsisten** - Gunakan naming convention yang konsisten
4. **Dokumentasi** - Dokumentasikan penggunaan tag dalam team

---

## Monitoring dan Maintenance

### Melihat Status Runner

#### Status Container

```bash
# Status container
docker compose ps

# Detail container
docker inspect gitlab-runner

# Resource usage real-time
docker stats gitlab-runner
```

#### Status Runner di GitLab

1. Buka GitLab UI
2. Navigate ke **Settings** > **CI/CD** > **Runners**
3. Lihat status runner (hijau = online, abu-abu = offline)

#### Logs

```bash
# Logs real-time
docker compose logs -f gitlab-runner

# Logs 100 baris terakhir
docker compose logs --tail=100 gitlab-runner

# Logs dengan timestamp
docker compose logs -f --timestamps gitlab-runner

# Logs hari ini
docker compose logs --since $(date -d "today" +%Y-%m-%d) gitlab-runner
```

#### Verifikasi Runner

```bash
# Verifikasi runner terdaftar
docker compose exec gitlab-runner gitlab-runner verify

# List semua runner
docker compose exec gitlab-runner gitlab-runner list

# Status runner
docker compose exec gitlab-runner gitlab-runner status
```

### Monitoring Resource

#### CPU dan Memory Usage

```bash
# Monitor real-time
docker stats gitlab-runner

# Dengan format custom
docker stats gitlab-runner --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"
```

#### Disk Usage

```bash
# Cache directory size
du -sh ~/gitlab-runner/cache

# Detail per subdirectory
du -h --max-depth=1 ~/gitlab-runner/cache

# Docker disk usage
docker system df
```

#### Network Usage

```bash
# Network stats
docker network inspect gitlab-runner-network

# Connection stats
docker compose exec gitlab-runner netstat -tuln
```

### Maintenance Rutin

#### Update Runner

```bash
# Pull image terbaru
docker compose pull

# Restart dengan image baru
docker compose up -d

# Verifikasi versi
docker compose exec gitlab-runner gitlab-runner --version
```

#### Restart Runner

```bash
# Restart container
docker compose restart

# Stop dan start
docker compose stop
docker compose start

# Recreate container
docker compose down
docker compose up -d
```

#### Membersihkan Cache

```bash
# Bersihkan cache lama (lebih dari 7 hari)
find ~/gitlab-runner/cache -type f -mtime +7 -delete

# Bersihkan semua cache
rm -rf ~/gitlab-runner/cache/*

# Bersihkan Docker cache
docker compose exec gitlab-runner gitlab-runner cache-clear
```

#### Membersihkan Docker Resources

```bash
# Bersihkan images tidak terpakai
docker image prune -a -f

# Bersihkan containers stopped
docker container prune -f

# Bersihkan volumes tidak terpakai
docker volume prune -f

# Bersihkan semua (hati-hati!)
docker system prune -a -f --volumes
```

#### Backup Konfigurasi

```bash
# Backup manual
tar -czf gitlab-runner-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  ~/gitlab-runner/config.toml \
  ~/gitlab-runner/docker-compose.yaml

# Restore
tar -xzf gitlab-runner-backup-YYYYMMDD-HHMMSS.tar.gz -C ~/
```

#### Backup Otomatis (Cron)

```bash
# Edit crontab
crontab -e

# Tambahkan baris ini untuk backup setiap hari jam 2 pagi
0 2 * * * tar -czf ~/backups/gitlab-runner-$(date +\%Y\%m\%d).tar.gz ~/gitlab-runner/config.toml ~/gitlab-runner/docker-compose.yaml
```

### Health Check

#### Script Health Check

```bash
#!/bin/bash
# health-check.sh

# Warna output
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m'

echo "=== GitLab Runner Health Check ==="

# Check container running
if docker compose ps | grep -q "Up"; then
    echo -e "${GREEN}✓ Container running${NC}"
else
    echo -e "${RED}✗ Container not running${NC}"
    exit 1
fi

# Check runner registered
if docker compose exec gitlab-runner gitlab-runner verify 2>&1 | grep -q "is alive"; then
    echo -e "${GREEN}✓ Runner registered and alive${NC}"
else
    echo -e "${RED}✗ Runner not registered or dead${NC}"
    exit 1
fi

# Check disk space
DISK_USAGE=$(df -h ~/gitlab-runner | awk 'NR==2 {print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -lt 80 ]; then
    echo -e "${GREEN}✓ Disk usage: ${DISK_USAGE}%${NC}"
else
    echo -e "${RED}✗ Disk usage high: ${DISK_USAGE}%${NC}"
fi

# Check memory usage
MEM_USAGE=$(docker stats gitlab-runner --no-stream --format "{{.MemPerc}}" | sed 's/%//')
if (( $(echo "$MEM_USAGE < 90" | bc -l) )); then
    echo -e "${GREEN}✓ Memory usage: ${MEM_USAGE}%${NC}"
else
    echo -e "${RED}✗ Memory usage high: ${MEM_USAGE}%${NC}"
fi

echo "=== Health Check Complete ==="
```

Jalankan:
```bash
chmod +x health-check.sh
./health-check.sh
```

---

## Troubleshooting

### Problem 1: Runner Tidak Muncul di GitLab

**Gejala:**
- Container berjalan tapi runner tidak terlihat di GitLab UI
- Status runner offline

**Penyebab:**
- Token salah atau expired
- URL GitLab salah
- Network issue

**Solusi:**

```bash
# 1. Verifikasi konfigurasi
cat config.toml | grep -E "url|token"

# 2. Cek logs untuk error
docker compose logs gitlab-runner | grep -i error

# 3. Verifikasi koneksi ke GitLab
docker compose exec gitlab-runner curl -I https://gitlab.com/

# 4. Re-register runner
docker compose exec gitlab-runner gitlab-runner verify

# 5. Restart runner
docker compose restart
```

### Problem 2: Job Stuck di Pending

**Gejala:**
- Job tidak pernah dieksekusi
- Status pending terus menerus

**Penyebab:**
- Tags tidak match
- Concurrent limit tercapai
- Runner offline

**Solusi:**

```bash
# 1. Cek tags di job vs runner
docker compose exec gitlab-runner gitlab-runner list

# 2. Cek concurrent jobs
docker compose exec gitlab-runner gitlab-runner list

# 3. Tingkatkan concurrent limit jika perlu
nano config.toml
# Ubah: concurrent = 10
docker compose restart

# 4. Cek runner status di GitLab UI
```

### Problem 3: Job Gagal dengan "Cannot Connect to Docker Daemon"

**Gejala:**
- Error: `Cannot connect to the Docker daemon`
- Job gagal saat mencoba pull image

**Penyebab:**
- Docker socket tidak ter-mount dengan benar
- Permission issue

**Solusi:**

```bash
# 1. Cek Docker socket
ls -l /var/run/docker.sock

# 2. Perbaiki permission
sudo chmod 666 /var/run/docker.sock

# 3. Restart Docker
sudo systemctl restart docker

# 4. Restart runner
docker compose restart

# 5. Verifikasi mount
docker compose exec gitlab-runner ls -l /var/run/docker.sock
```

### Problem 4: Out of Memory Error

**Gejala:**
- Job killed dengan error OOM
- Container restart sendiri

**Penyebab:**
- Job menggunakan memory lebih dari limit
- Terlalu banyak job bersamaan

**Solusi:**

```bash
# 1. Monitor memory usage
docker stats gitlab-runner

# 2. Kurangi concurrent jobs
nano config.toml
# Ubah: concurrent = 6
docker compose restart

# 3. Tingkatkan memory per job
nano config.toml
# Ubah: memory = "3g"
docker compose restart

# 4. Bersihkan Docker cache
docker system prune -a -f
```

### Problem 5: Disk Space Full

**Gejala:**
- Error: `no space left on device`
- Job gagal saat download/build

**Penyebab:**
- Cache terlalu besar
- Docker images menumpuk
- Logs terlalu besar

**Solusi:**

```bash
# 1. Cek disk usage
df -h
du -sh ~/gitlab-runner/*

# 2. Bersihkan cache
rm -rf ~/gitlab-runner/cache/*

# 3. Bersihkan Docker
docker system prune -a -f --volumes

# 4. Bersihkan logs
docker compose logs --tail=0 gitlab-runner

# 5. Bersihkan old images
docker image prune -a --filter "until=168h" -f
```

### Problem 6: Job Timeout

**Gejala:**
- Job dibatalkan setelah waktu tertentu
- Error: `Job's log exceeded limit`

**Penyebab:**
- Session timeout terlalu pendek
- Job memang terlalu lama

**Solusi:**

```bash
# 1. Tingkatkan session timeout
nano config.toml
# Ubah: session_timeout = 7200 (2 jam)
docker compose restart

# 2. Atau set timeout di .gitlab-ci.yml
# timeout: 2h
```

### Problem 7: Network Connectivity Issues

**Gejala:**
- Tidak bisa pull Docker images
- Tidak bisa akses internet dari job

**Penyebab:**
- Network configuration salah
- DNS issue
- Firewall blocking

**Solusi:**

```bash
# 1. Test koneksi dari runner
docker compose exec gitlab-runner ping -c 4 8.8.8.8
docker compose exec gitlab-runner curl -I https://hub.docker.com

# 2. Cek DNS
docker compose exec gitlab-runner cat /etc/resolv.conf

# 3. Restart network
docker network rm gitlab-runner-network
docker compose up -d

# 4. Cek firewall
sudo ufw status
```

### Problem 8: Permission Denied

**Gejala:**
- Error: `permission denied` saat build
- Tidak bisa write ke cache

**Penyebab:**
- File permission salah
- Volume mount permission

**Solusi:**

```bash
# 1. Perbaiki permission cache
sudo chown -R 999:999 ~/gitlab-runner/cache

# 2. Perbaiki permission config
sudo chmod 600 ~/gitlab-runner/config.toml

# 3. Restart runner
docker compose restart
```

### Problem 9: Runner Version Mismatch

**Gejala:**
- Warning tentang versi runner
- Feature tidak tersedia

**Solusi:**

```bash
# 1. Cek versi runner
docker compose exec gitlab-runner gitlab-runner --version

# 2. Update ke versi terbaru
docker compose pull
docker compose up -d

# 3. Atau gunakan versi specific
nano docker-compose.yaml
# Ubah: image: gitlab/gitlab-runner:ubuntu-v18.5.0
```

### Problem 10: Certificate Errors

**Gejala:**
- SSL certificate verification error
- TLS handshake failure

**Solusi:**

```bash
# 1. Disable TLS verify (tidak recommended untuk production)
nano config.toml
# Ubah: tls_verify = false

# 2. Atau tambahkan custom CA certificate
docker compose exec gitlab-runner \
  cp /path/to/ca.crt /etc/gitlab-runner/certs/

# 3. Restart runner
docker compose restart
```

---

## Best Practices

### Security

#### 1. Jangan Gunakan Privileged Mode

```toml
# ❌ Hindari
privileged = true

# ✅ Gunakan
privileged = false
```

**Alasan:** Privileged mode memberikan akses penuh ke host system, berbahaya jika job dieksekusi dari untrusted source.

#### 2. Batasi Resource per Job

```toml
# Selalu set resource limits
cpus = "1"
memory = "2g"
memory_swap = "2g"
```

**Alasan:** Mencegah single job menghabiskan semua resource dan membuat runner down.

#### 3. Gunakan Non-Root User

```yaml
# Di .gitlab-ci.yml
variables:
  DOCKER_USER: "gitlab-runner"
```

#### 4. Rotate Token Secara Berkala

```bash
# Setiap 3-6 bulan, generate token baru dan update config
nano config.toml
docker compose restart
```

#### 5. Monitor Access Logs

```bash
# Regularly check logs untuk aktivitas mencurigakan
docker compose logs gitlab-runner | grep -i "error\|failed\|unauthorized"
```

#### 6. Gunakan Private Registry

```yaml
# Di .gitlab-ci.yml
image: registry.example.com/myapp:latest
```

#### 7. Enable Rate Limiting

```toml
# Di config.toml
[runners.docker]
  # Limit concurrent pulls
  pull_policy = ["if-not-present"]
```

### Performance

#### 1. Optimalkan Concurrent Jobs

**Formula:** `concurrent = (Total CPU / CPU per job) - 1`

Untuk 8 CPU dengan 1 CPU per job:
```toml
concurrent = 7  # Bukan 8, sisakan 1 untuk system
```

#### 2. Gunakan Local Cache

```toml
[runners.cache]
  Type = "local"
  Path = "/cache"
  Shared = true
```

**Benefit:** Mempercepat build dengan cache dependency.

#### 3. Optimalkan Pull Policy

```toml
pull_policy = ["if-not-present"]
```

**Benefit:** Tidak re-download image jika sudah ada.

#### 4. Gunakan Shared Memory

```toml
shm_size = 2147483648  # 2GB
```

**Benefit:** Penting untuk aplikasi yang menggunakan shared memory (database, browsers).

#### 5. Cleanup Rutin

```bash
# Buat cron job untuk cleanup otomatis
0 2 * * * docker system prune -a -f
0 3
