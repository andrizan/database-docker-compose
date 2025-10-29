# Next.js CI/CD & PM2 Configuration

Konfigurasi untuk deployment Next.js menggunakan GitLab CI/CD dan PM2 process manager dengan standalone build mode.

## 📁 File yang Tersedia

- **`.gitlab-ci.yml`**: Pipeline CI/CD untuk build dan deploy Next.js application
- **`ecosystem.config.js`**: Konfigurasi PM2 untuk production deployment

## 🚀 GitLab CI/CD Pipeline

Pipeline terdiri dari 5 stage:

### 1. Dependencies Stage
- **`install:dependencies`**: Install npm dependencies menggunakan `npm ci`
- **`download-secret-var`**: Download file `.env` dari GitLab Secure Files

### 2. Lint Stage
- **`lint:format`**: Jalankan format checker
- **`lint:check`**: Jalankan linter (ESLint)

### 3. Build Stage
- **`build`**: Build Next.js dengan standalone output
  - Output standalone build ke `.next/standalone/`
  - Copy static files dan public folder
  - Generate artifacts untuk deployment

### 4. Deploy Stage
- **`deploy`**: Deploy ke production server menggunakan rsync via SSH
  - Upload build artifacts ke server
  - Restart/reload aplikasi dengan PM2

### 5. Release Stage
- **`release:create`**: Buat GitLab Release untuk semantic versioning tags

## ⚙️ Konfigurasi Required

### GitLab CI/CD Variables

Tambahkan variables berikut di GitLab CI/CD Settings:

| Variable | Description | Example |
|----------|-------------|---------|
| `SSH_PRIVATE_KEY` | Private SSH key (base64 encoded) | `base64 -w0 < ~/.ssh/id_rsa` |
| `SSH_HOST` | Target server hostname/IP | `192.168.1.100` |
| `SSH_USER` | SSH user untuk deployment | `deployer` |
| `DEPLOY_PATH` | Path deployment di server | `/var/www/cms` |

### GitLab Secure Files

Upload file `.env` ke GitLab Project Settings > CI/CD > Secure Files untuk digunakan saat build dan deploy.

## 📦 PM2 Configuration

File `ecosystem.config.js` mengkonfigurasi PM2 dengan fitur:

### Cluster Mode
```javascript
instances: 2,
exec_mode: "cluster"
```
- Menjalankan 2 instance untuk load balancing
- Auto-restart jika aplikasi crash

### Memory Management
```javascript
max_memory_restart: "512M",
node_args: "--max-old-space-size=512"
```
- Auto-restart jika memory usage > 512MB
- Limit heap memory Node.js ke 512MB

### Logging
```javascript
error_file: "./logs/web-error.log",
out_file: "./logs/web-out.log"
```
- Error dan output logs disimpan di folder `logs/`

### Environment Variables
```javascript
env_production: {
  NODE_ENV: "production",
  PORT: 3000,
  HOSTNAME: "0.0.0.0"
}
```

## 🏗️ Next.js Configuration Requirements

Untuk menggunakan standalone build mode, tambahkan di `next.config.js`:

```javascript
module.exports = {
  output: 'standalone',
  // ... config lainnya
}
```

## 📋 Cara Menggunakan

### 1. Setup Project Next.js

Pastikan project Next.js memiliki:
- `next.config.js` dengan `output: 'standalone'`
- Script npm yang diperlukan di `package.json`:
  ```json
  {
    "scripts": {
      "dev": "next dev",
      "build": "next build",
      "start": "next start",
      "lint": "next lint",
      "format": "prettier --check ."
    }
  }
  ```

### 2. Copy Files

Copy kedua file ini ke root directory project Next.js:
```bash
cp .gitlab-ci.yml /path/to/your/nextjs-project/
cp ecosystem.config.js /path/to/your/nextjs-project/
```

### 3. Setup GitLab

1. Push project ke GitLab repository
2. Tambahkan CI/CD variables (SSH_PRIVATE_KEY, SSH_HOST, SSH_USER)
3. Upload file `.env` ke Secure Files
4. Setup GitLab Runner dengan tags: `install-autoscale`, `test-autoscale`, `build-autoscale`, `deploy-autoscale`, `release-autoscale`

### 4. Setup Production Server

Install dependencies di server:
```bash
# Install Node.js 22
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install PM2
sudo npm install -g pm2

# Setup PM2 startup
pm2 startup
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u $USER --hp /home/$USER

# Buat deployment directory
sudo mkdir -p /var/www/cms
sudo chown -R $USER:$USER /var/www/cms
```

### 5. Deployment

Pipeline akan otomatis berjalan untuk:
- **Merge Request**: Lint dan build
- **Push ke main branch**: Lint dan build
- **Tag semantic version** (e.g., `v1.0.0`): Lint, build, deploy, dan release

Untuk trigger deployment:
```bash
# Create and push semantic version tag
git tag v1.0.0
git push origin v1.0.0
```

## 🔄 PM2 Commands

Setelah deployment, manage aplikasi dengan PM2:

```bash
# Check status
pm2 status

# View logs
pm2 logs cms

# Restart
pm2 restart cms

# Reload (zero-downtime)
pm2 reload cms

# Stop
pm2 stop cms

# Monitor
pm2 monit
```

## 📊 Pipeline Triggers

| Event | Stages Run | Deploy? |
|-------|-----------|---------|
| Merge Request | dependencies, lint, build | ❌ |
| Push to main | dependencies, lint, build | ❌ |
| Tag `v*.*.*` | dependencies, lint, build, deploy, release | ✅ |

## 🔧 Customization

### Mengubah Jumlah Instance PM2
Edit `ecosystem.config.js`:
```javascript
instances: 4, // Ubah sesuai CPU cores
```

### Mengubah Port
Edit `ecosystem.config.js`:
```javascript
env_production: {
  PORT: 8080, // Port yang diinginkan
}
```

### Mengubah Deployment Path
Update variable `DEPLOY_PATH` di GitLab CI/CD atau edit di file:
```yaml
variables:
  DEPLOY_PATH: /var/www/custom-path
```

## 🐛 Troubleshooting

### Build Gagal
- Pastikan semua dependencies terinstall
- Check error di GitLab CI/CD pipeline logs
- Verify `.env` file di Secure Files

### Deployment Gagal
- Verify SSH credentials (SSH_PRIVATE_KEY, SSH_HOST, SSH_USER)
- Check SSH access ke server: `ssh user@host`
- Verify DEPLOY_PATH exists dan writable

### PM2 Tidak Start
- Check logs: `pm2 logs cms`
- Verify Node.js version: `node --version`
- Check PORT tidak digunakan proses lain: `lsof -i :3000`

## 📝 Notes

- Standalone build menghasilkan aplikasi self-contained yang tidak memerlukan `node_modules` di production
- PM2 cluster mode meningkatkan availability dan performance
- Auto-restart melindungi aplikasi dari crash
- Log rotation dihandle otomatis oleh PM2
