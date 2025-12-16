# Tata Cara Setup Redis dengan Docker Compose

Dokumen ini menjelaskan langkah-langkah untuk setup Redis menggunakan Docker Compose dengan network `infra_net`, nginx reverse proxy, dan Cloudflare Tunnel.

## Prasyarat

1. **Docker** terinstall (versi 29.1.2 atau lebih baru)
2. **Docker Compose** terinstall
3. **Network `infra_net`** sudah dibuat sebelumnya
4. **Nginx** reverse proxy yang terhubung ke `infra_net`
5. **Cloudflare Tunnel** untuk akses eksternal (opsional)

## Langkah-langkah Setup

### 1. Verifikasi Docker dan Docker Compose

Pastikan Docker dan Docker Compose sudah terinstall:

```bash
docker --version
docker compose version
```

### 2. Verifikasi Network infra_net

Pastikan network `infra_net` sudah ada:

```bash
docker network inspect infra_net
```

Jika network belum ada, buat terlebih dahulu:

```bash
docker network create infra_net
```

Verifikasi network sudah dibuat:

```bash
docker network ls | grep infra_net
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
- Menghubungkan ke network `infra_net`

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
# Dari container lain di infra_net
redis-cli -h redis ping
```

### 9. Konfigurasi Nginx (Opsional)

Jika ingin mengakses Redis melalui nginx reverse proxy, tambahkan konfigurasi di nginx:

```nginx
# Contoh konfigurasi nginx untuk Redis (stream proxy)
stream {
    upstream redis_backend {
        server redis:6379;
    }
    
    server {
        listen 6379;
        proxy_pass redis_backend;
        proxy_timeout 1s;
        proxy_responses 1;
        error_log /var/log/nginx/redis_error.log;
    }
}
```

**Catatan**: Konfigurasi di atas hanya contoh. Sesuaikan dengan kebutuhan dan arsitektur nginx Anda.

### 10. Konfigurasi Cloudflare Tunnel (Opsional)

Untuk akses eksternal melalui Cloudflare Tunnel, konfigurasi tunnel Anda untuk mengarahkan ke nginx atau langsung ke Redis (jika diizinkan).

**Peringatan Keamanan**: Pastikan Redis hanya diakses melalui jaringan yang aman. Jangan expose Redis langsung ke internet tanpa autentikasi yang kuat.

## Konfigurasi Tambahan

### Mengakses Redis dari Aplikasi

Gunakan hostname `redis` dan port `6379` untuk mengakses dari container lain di network `infra_net`:

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

Redis tidak di-expose langsung ke host untuk keamanan. Akses eksternal dilakukan melalui:

- **Nginx Reverse Proxy**: Konfigurasi nginx untuk proxy ke `redis:6379` di network `infra_net`
- **Cloudflare Tunnel**: Untuk akses dari internet melalui Cloudflare

Jika perlu akses langsung dari host untuk testing/development, tambahkan port mapping di `docker-compose.yml`:

```yaml
ports:
  - "127.0.0.1:6379:6379"  # Hanya accessible dari localhost
```

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

2. Pastikan network `infra_net` sudah ada:

```bash
docker network inspect infra_net
```

3. Pastikan port 6379 tidak digunakan aplikasi lain:

```bash
sudo netstat -tulpn | grep 6379
```

### Tidak Bisa Koneksi dari Container Lain

1. Pastikan container lain juga menggunakan network `infra_net`
2. Gunakan hostname `redis` (bukan `localhost`)
3. Cek apakah Redis container berjalan:

```bash
docker ps | grep redis
```

4. Verifikasi network connection:

```bash
docker network inspect infra_net | grep redis
```

### Permission Denied pada Volume

Jika ada masalah permission:

```bash
sudo chown -R $USER:$USER ./redis-data
```

## Informasi Teknis

- **Image**: `redis:7-alpine` (versi 7, Alpine Linux)
- **Port**: `6379` (default Redis port, internal only)
- **Volume**: `redis-data` (persistent storage)
- **Network**: `infra_net` (external)
- **Health Check**: Otomatis setiap 10 detik
- **Reverse Proxy**: Nginx (jika dikonfigurasi)
- **External Access**: Cloudflare Tunnel (jika dikonfigurasi)

## Keamanan

1. **Network Isolation**: Redis hanya accessible dari network `infra_net`, tidak di-expose ke host
2. **Password**: Aktifkan password di `redis.conf` jika Redis akan diakses melalui nginx/Cloudflare Tunnel
3. **Nginx**: Gunakan nginx sebagai reverse proxy untuk kontrol akses yang lebih baik
4. **Cloudflare Tunnel**: Gunakan Cloudflare Tunnel untuk akses eksternal yang aman
5. **Backup**: Lakukan backup data secara berkala
6. **Firewall**: Pastikan port 6379 tidak terbuka langsung ke internet

## Support

Jika ada masalah, cek:
- Logs container: `docker compose logs redis`
- Status container: `docker compose ps`
- Network: `docker network inspect infra_net`
- Nginx logs (jika menggunakan nginx): `docker logs <nginx-container>`
- Cloudflare Tunnel status (jika menggunakan tunnel)

