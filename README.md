# Instalasi Outline Wiki dengan Docker Compose (Mode Lokal)

Panduan ini memasang [Outline](https://www.getoutline.com/) (knowledge base / wiki tim) beserta PostgreSQL dan Redis di satu komputer/server menggunakan Docker Compose. Aksesnya langsung lewat port, tanpa domain dan tanpa SSL. Contohnya `http://localhost:8070` atau `http://192.168.1.10:8070`.

Referensi resmi: <https://docs.getoutline.com/s/hosting/doc/docker-7pfeLP5a8t>

---

## 1. Arsitektur

```
Browser ──► http://<host>:8070 ──► outline ──► postgres:5432  (data dokumen)
                                      │    └─► redis:6379     (cache & antrian)
                                      ▼
                          volume storage-data (lampiran & gambar)
```

| Service    | Image                                       | Port ke host |
|------------|---------------------------------------------|--------------|
| `outline`  | `docker.getoutline.com/outlinewiki/outline` | **8070**     |
| `postgres` | `postgres:18`                               | tidak dibuka |
| `redis`    | `redis:7-alpine`                            | tidak dibuka |

Volume yang dibuat: `database-data`, `storage-data`, `redis-data`.

## 2. Prasyarat

- Linux, Windows (Docker Desktop/WSL2), atau macOS dengan RAM minimal **2 GB**.
- **Docker Engine** dan **Docker Compose v2** (cek dengan `docker compose version`).
- Port **8070** belum dipakai aplikasi lain.
- Satu **penyedia login**. Outline **tidak** punya login username/password bawaan (lihat bagian 5).

## 3. Struktur File

```
outline-docker/
├── docker-compose.yml
├── docker.env.example   # template konfigurasi
├── docker.env           # konfigurasi Anda (dibuat dari template)
└── README.md
```

## 4. Langkah Instalasi

### 4.1 Siapkan konfigurasi

```bash
cd outline-docker
cp docker.env.example docker.env
```

### 4.2 Buat secret

```bash
openssl rand -hex 32   # untuk SECRET_KEY
openssl rand -hex 32   # untuk UTILS_SECRET
openssl rand -hex 16   # untuk password database
```

### 4.3 Edit `docker.env`

| Variabel            | Nilai                                                    |
|---------------------|----------------------------------------------------------|
| `URL`               | `http://localhost:8070` atau `http://<IP-server>:8070`   |
| `SECRET_KEY`        | hasil `openssl rand -hex 32`                             |
| `UTILS_SECRET`      | hasil `openssl rand -hex 32`                             |
| `POSTGRES_PASSWORD` | password database                                        |
| `DATABASE_URL`      | `postgres://outline:<password>@postgres:5432/outline`    |
| `FORCE_HTTPS`       | `false` (wajib untuk HTTP)                               |
| Provider login      | lihat bagian 5                                           |

> ⚠️ **`URL` harus sama persis dengan alamat yang diketik di browser.** Jika `URL=http://localhost:8070` tetapi Anda membuka `http://192.168.1.10:8070`, login akan gagal atau redirect ke alamat yang salah.
>
> ⚠️ Password di `POSTGRES_PASSWORD` dan `DATABASE_URL` harus sama. Gunakan password heksadesimal agar tidak perlu URL-encode.

### 4.4 Pilih cakupan akses (di `docker-compose.yml`)

```yaml
ports:
  - "8070:3000"            # bisa diakses komputer lain di jaringan LAN
  # - "127.0.0.1:8070:3000"  # hanya dari komputer ini
```

Angka kiri (8070) adalah port di host, angka kanan (3000) adalah port internal Outline dan tidak perlu diubah. Jika ingin port lain, ubah angka kiri saja lalu sesuaikan `URL`.

### 4.5 Jalankan

```bash
docker compose pull
docker compose up -d
docker compose ps
docker compose logs -f outline
```

Migrasi database berjalan otomatis saat container start. Tunggu sampai log tidak menampilkan error koneksi database/redis, lalu buka alamat di `URL`. **User pertama yang login otomatis menjadi Admin.**

Untuk akses dari komputer lain di LAN, buka port di firewall server. Contoh di Ubuntu: `sudo ufw allow 8070/tcp`.

## 5. Penyedia Login untuk Mode Lokal

Karena tanpa domain dan HTTPS, tidak semua provider bisa dipakai:

| Provider        | Akses `localhost` | Akses via IP LAN | Catatan |
|-----------------|:-----------------:|:----------------:|---------|
| Google          | ✅ | ❌ | Google hanya menerima redirect `http://localhost`, tidak menerima IP. |
| Slack           | ❌ | ❌ | Slack mewajibkan redirect HTTPS. |
| Microsoft Entra | ✅ | ❌ | `http` hanya diizinkan untuk `localhost`. |
| OIDC (Keycloak, Authentik, dll.) | ✅ | ✅ | **Paling cocok untuk LAN.** IdP bisa dijalankan di jaringan lokal juga. |

Callback URL formatnya `<URL>/auth/<provider>.callback`. Contoh:

- `http://localhost:8070/auth/google.callback`
- `http://192.168.1.10:8070/auth/oidc.callback`

**Contoh Google (khusus localhost):**
1. Buka <https://console.cloud.google.com/apis/credentials>, lalu pilih *Create Credentials* → *OAuth client ID* → *Web application*.
2. Isi *Authorized redirect URI* dengan `http://localhost:8070/auth/google.callback`.
3. Salin Client ID & Secret ke `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET`.

**Contoh OIDC dengan Keycloak (untuk akses LAN):**
1. Di Keycloak, buat client `outline` (Client authentication: ON).
2. Isi *Valid redirect URIs* dengan `http://192.168.1.10:8070/auth/oidc.callback`.
3. Isi `OIDC_*` di `docker.env`, dengan endpoint mengarah ke Keycloak, misalnya `http://192.168.1.10:8080/realms/<realm>/protocol/openid-connect/auth`.

## 6. Operasional

### Perintah umum

```bash
docker compose ps                 # status
docker compose logs -f outline    # log aplikasi
docker compose restart outline    # restart setelah ubah docker.env
docker compose down               # hentikan (data tetap aman di volume)
```

> Setelah mengubah `docker.env`, jalankan `docker compose up -d` agar container dibuat ulang dengan nilai baru.

### Backup

```bash
# Database
docker compose exec -T postgres pg_dump -U outline -d outline -Fc > outline-$(date +%F).dump

# Lampiran
docker run --rm -v outline_storage-data:/data -v "$PWD":/backup alpine \
  tar czf /backup/outline-files-$(date +%F).tar.gz -C /data .
```

### Restore

```bash
docker compose stop outline
docker compose exec -T postgres pg_restore -U outline -d outline --clean --if-exists < outline-YYYY-MM-DD.dump
docker run --rm -v outline_storage-data:/data -v "$PWD":/backup alpine \
  sh -c "cd /data && tar xzf /backup/outline-files-YYYY-MM-DD.tar.gz"
docker compose start outline
```

### Update versi

1. Backup database.
2. Ambil image terbaru dengan `docker compose pull outline`, atau ubah tag versi di `docker-compose.yml`.
3. Jalankan `docker compose up -d`. Migrasi berjalan otomatis.

> Disarankan **pin versi** image (mis. `outlinewiki/outline:1.2.0`). Daftar rilis ada di <https://github.com/outline/outline/releases>.

## 7. Troubleshooting

| Gejala | Penyebab & Solusi |
|---|---|
| Browser dialihkan ke `https://...` | `FORCE_HTTPS` belum `false`. Ubah, lalu jalankan `docker compose up -d`. |
| Tidak ada tombol login | Belum ada provider login yang diisi di `docker.env`. |
| `redirect_uri_mismatch` / login kembali ke alamat lain | `URL` tidak sama dengan alamat di browser atau dengan callback di console provider. |
| Tidak bisa dibuka dari komputer lain | Port memakai `127.0.0.1:8070:3000`, atau firewall memblokir port 8070. |
| `port is already allocated` | Port 8070 sudah dipakai. Ganti misalnya ke `"8080:3000"` dan sesuaikan `URL`. |
| `password authentication failed` | Password `DATABASE_URL` ≠ `POSTGRES_PASSWORD`. Jika volume sudah terlanjur dibuat dengan password lama, ubah password di Postgres atau hapus volume `outline_database-data` (data hilang). |
| Upload gambar gagal (`EACCES`) | Jalankan `docker compose exec -u root outline chown -R 1001:1001 /var/lib/outline/data`. |

## 8. Jika Nanti Ingin Dipublikasikan

Tambahkan reverse proxy dengan SSL (https-portal, Nginx, Caddy, atau Traefik) di depan port 8070, ubah `URL` ke `https://domain-anda`, lalu perbarui callback URL di provider login. Lihat <https://docs.getoutline.com/doc/ssl-pzk7WO8d1n>.

## 9. Referensi

- Docker: <https://docs.getoutline.com/s/hosting/doc/docker-7pfeLP5a8t>
- Contoh variabel lengkap: <https://github.com/outline/outline/blob/main/.env.sample>
- Backup: <https://docs.getoutline.com/doc/backups-KZtPOADCHG>
- OIDC: <https://docs.getoutline.com/s/hosting/doc/oidc-8CPBm6uC0I>