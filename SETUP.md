# Tata Cara Setup Redis dengan Docker Compose

Dokumen ini menjelaskan langkah-langkah untuk setup Redis menggunakan Docker Compose dengan network Traefik di server Ubuntu.

## Prasyarat

1. **Docker** terinstall (versi 29.1.2 atau lebih baru)
2. **Docker Compose** terinstall
3. **Network Traefik** sudah dibuat sebelumnya

## Langkah-langkah Setup

### 1. Verifikasi Docker dan Docker Compose

Pastikan Docker dan Docker Compose sudah terinstall:

```bash
docker --version
docker compose version
```

### 2. Buat Network Traefik (jika belum ada)

Jika network `traefik-network` belum ada, buat terlebih dahulu:

```bash
docker network create traefik-network
```

Verifikasi network sudah dibuat:

```bash
docker network ls | grep traefik-network
```

### 3. Persiapkan File Konfigurasi

Pastikan file-file berikut ada di direktori project:

- `docker-compose.yml` - File konfigurasi Docker Compose
- `redis.conf` - File konfigurasi Redis

### 4. Review Konfigurasi Redis (Opsional)

Edit file `redis.conf` sesuai kebutuhan Anda:

- **maxmemory**: Sesuaikan dengan kapasitas RAM yang tersedia
- **maxmemory-policy**: Pilih policy yang sesuai (allkeys-lru, volatile-lru, dll)
- **protected-mode**: Set ke `yes` jika ingin lebih aman (perlu password)

### 5. Jalankan Redis Container

Jalankan Redis menggunakan Docker Compose:

```bash
docker compose up -d
```

Perintah ini akan:
- Pull image Redis (jika belum ada)
- Membuat volume untuk data persistence
- Menjalankan container Redis di background
- Menghubungkan ke network `traefik-network`

### 6. Verifikasi Container Berjalan

Cek status container:

```bash
docker compose ps
```

Atau:

```bash
docker ps | grep redis
```

### 7. Test Koneksi Redis

Test koneksi dari host:

```bash
docker exec -it redis redis-cli ping
```

Output yang diharapkan: `PONG`

### 8. Test Koneksi dari Container Lain

Untuk test dari container lain di network yang sama:

```bash
# Dari container lain di traefik-network
redis-cli -h redis ping
```

## Konfigurasi Tambahan

### Mengakses Redis dari Aplikasi

Gunakan hostname `redis` dan port `6379` untuk mengakses dari container lain di network `traefik-network`:

```python
# Contoh Python
import redis
r = redis.Redis(host='redis', port=6379, db=0)
```

```javascript
// Contoh Node.js
const redis = require('redis');
const client = redis.createClient({
  host: 'redis',
  port: 6379
});
```

### Mengakses dari Host (External)

Untuk mengakses dari host Ubuntu atau dari luar Docker:

- **Host**: `localhost` atau IP server
- **Port**: `6379`

### Menambahkan Password (Opsional)

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

## Perintah Berguna

### Melihat Logs

```bash
docker compose logs -f redis
```

### Stop Redis

```bash
docker compose stop redis
```

### Start Redis

```bash
docker compose start redis
```

### Restart Redis

```bash
docker compose restart redis
```

### Stop dan Hapus Container (Data Tetap Tersimpan)

```bash
docker compose down
```

### Stop dan Hapus Container + Volume (Hapus Data)

```bash
docker compose down -v
```

### Masuk ke Redis CLI

```bash
docker exec -it redis redis-cli
```

### Backup Data Redis

```bash
docker exec redis redis-cli SAVE
docker cp redis:/data/dump.rdb ./backup-$(date +%Y%m%d).rdb
```

### Restore Data Redis

```bash
docker cp ./backup-YYYYMMDD.rdb redis:/data/dump.rdb
docker compose restart redis
```

## Troubleshooting

### Container Tidak Bisa Start

1. Cek logs:

```bash
docker compose logs redis
```

2. Pastikan network `traefik-network` sudah ada:

```bash
docker network inspect traefik-network
```

3. Pastikan port 6379 tidak digunakan aplikasi lain:

```bash
sudo netstat -tulpn | grep 6379
```

### Tidak Bisa Koneksi dari Container Lain

1. Pastikan container lain juga menggunakan network `traefik-network`
2. Gunakan hostname `redis` (bukan `localhost`)
3. Cek apakah Redis container berjalan:

```bash
docker ps | grep redis
```

### Permission Denied pada Volume

Jika ada masalah permission:

```bash
sudo chown -R $USER:$USER ./redis-data
```

## Informasi Teknis

- **Image**: `redis:7-alpine` (versi 7, Alpine Linux)
- **Port**: `6379` (default Redis port)
- **Volume**: `redis-data` (persistent storage)
- **Network**: `traefik-network` (external)
- **Health Check**: Otomatis setiap 10 detik

## Keamanan

1. **Firewall**: Pastikan port 6379 tidak terbuka ke internet jika tidak diperlukan
2. **Password**: Aktifkan password jika Redis akan diakses dari luar
3. **Network**: Gunakan Docker network untuk isolasi
4. **Backup**: Lakukan backup data secara berkala

## Support

Jika ada masalah, cek:
- Logs container: `docker compose logs redis`
- Status container: `docker compose ps`
- Network: `docker network inspect traefik-network`

