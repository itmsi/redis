# Redis Docker Setup

Setup Redis menggunakan Docker Compose dengan network Traefik untuk server Ubuntu.

## 🚀 Quick Start

```bash
# 1. Pastikan network traefik-network sudah ada
docker network create traefik-network

# 2. Jalankan Redis
docker compose up -d

# 3. Test koneksi
docker exec -it redis redis-cli ping
```

## 📋 Prasyarat

- Docker versi 29.1.2 atau lebih baru
- Docker Compose
- Network `traefik-network` (akan dibuat otomatis jika belum ada)

## 📁 Struktur File

```
redis/
├── docker-compose.yml    # Konfigurasi Docker Compose
├── redis.conf            # Konfigurasi Redis
├── README.md             # Dokumentasi utama (file ini)
└── SETUP.md              # Panduan setup lengkap
```

## 🔧 Konfigurasi

### Docker Compose

- **Image**: `redis:7-alpine`
- **Container Name**: `redis`
- **Port**: `6379`
- **Network**: `traefik-network` (external)
- **Volume**: `redis-data` (persistent storage)
- **Health Check**: Enabled

### Redis Configuration

- **Bind**: `0.0.0.0` (semua interface)
- **Protected Mode**: `no` (untuk akses dari container lain)
- **Persistence**: RDB + AOF enabled
- **Max Memory**: `256mb` (dapat disesuaikan)
- **Memory Policy**: `allkeys-lru`

## 🔌 Mengakses Redis

### Dari Container Lain (Internal)

Gunakan hostname `redis` dan port `6379`:

```python
# Python
import redis
r = redis.Redis(host='redis', port=6379, db=0)
```

```javascript
// Node.js
const redis = require('redis');
const client = redis.createClient({
  host: 'redis',
  port: 6379
});
```

### Dari Host (External)

- **Host**: `localhost` atau IP server
- **Port**: `6379`

```bash
redis-cli -h localhost -p 6379
```

## 📝 Perintah Berguna

```bash
# Melihat logs
docker compose logs -f redis

# Stop Redis
docker compose stop redis

# Start Redis
docker compose start redis

# Restart Redis
docker compose restart redis

# Stop dan hapus container (data tetap tersimpan)
docker compose down

# Masuk ke Redis CLI
docker exec -it redis redis-cli

# Cek status container
docker compose ps
```

## 🔐 Keamanan

### Menambahkan Password

1. Edit `redis.conf`:
```conf
requirepass your_strong_password_here
```

2. Restart container:
```bash
docker compose restart redis
```

3. Gunakan password saat koneksi:
```bash
redis-cli -a your_strong_password_here
```

## 💾 Backup & Restore

### Backup

```bash
docker exec redis redis-cli SAVE
docker cp redis:/data/dump.rdb ./backup-$(date +%Y%m%d).rdb
```

### Restore

```bash
docker cp ./backup-YYYYMMDD.rdb redis:/data/dump.rdb
docker compose restart redis
```

## 🐛 Troubleshooting

### Container tidak bisa start

```bash
# Cek logs
docker compose logs redis

# Pastikan network ada
docker network inspect traefik-network

# Cek port tidak digunakan
sudo netstat -tulpn | grep 6379
```

### Tidak bisa koneksi dari container lain

1. Pastikan container lain juga di network `traefik-network`
2. Gunakan hostname `redis` (bukan `localhost`)
3. Cek status container: `docker ps | grep redis`

## 📚 Dokumentasi Lengkap

Untuk panduan setup lebih detail, lihat [SETUP.md](./SETUP.md)

## 📄 Lisensi

Proyek ini menggunakan image Redis resmi dari Docker Hub.

## 🔗 Referensi

- [Redis Documentation](https://redis.io/documentation)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Redis Docker Image](https://hub.docker.com/_/redis)

