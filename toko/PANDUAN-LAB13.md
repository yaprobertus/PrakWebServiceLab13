# Panduan Lab 13: Deployment Microservices (Docker Compose)

Semua perintah dijalankan dari folder `toko` (tempat `docker-compose.yml` berada), kecuali disebut lain.

## Yang sudah disiapkan di repository

| Langkah lab | Hasil di repository |
|---|---|
| 01 Konfigurasi ke environment | `gateway/gateway.js` membaca `PRODUCT_SERVICE_URL` dan `ORDER_SERVICE_URL` (default `localhost` dipertahankan); order-service dan notification-service sudah membaca `PRODUCT_SERVICE_URL` dan `AMQP_URL` |
| 02 Dockerfile dan .dockerignore | `product-service/`, `order-service/`, `notification-service/`, `gateway/` masing-masing punya `Dockerfile` dan `.dockerignore` (isi sama: node_modules, .git, npm-debug.log, .env) |
| 03 docker-compose.yml | `docker-compose.yml` (rabbitmq, product-service, order-service, notification-service, gateway; healthcheck dan `depends_on: service_healthy`; hanya :8080 dan :15672 dibuka ke host) |
| 06 CI | `.github/workflows/ci.yml` di akar repository memakai Compose (job `toko` dan `proyek-kelompok`) |
| 07 README | bagian "Menjalankan dengan Docker Compose" di `README.md` (akar) dan `toko/README.md` |
| Penerapan proyek kelompok | `proyek-kelompok/docker-compose.yml` + Dockerfile di `jadwal-service/`, `booking-service/`, `gateway/` |
| Refleksi | `docs/refleksi-lab13.md` |

Catatan API key: `API_KEY` sengaja tidak ditulis di compose. Gateway membacanya dari `gateway/.env`, dan `npm test` membaca dari berkas yang sama (di CI berkas dibuat oleh langkah "Siapkan API key"). Jika `gateway/.env` belum ada, jalankan `npm run start:all` sekali lalu hentikan (file dibuat otomatis), atau salin `gateway/.env.example` menjadi `gateway/.env`.

## Langkah 04: jalankan dan uji

```bash
docker rm -f rabbit              # bila container RabbitMQ Lab 10 masih ada
docker compose up --build        # terminal 1
docker compose ps                # terminal 2: semua Up / healthy
npm test                         # 11 test lulus
docker compose logs order-service product-service notification-service
```

Buka http://localhost:15672 (guest/guest). Kirim satu order lewat `node lacak-order.js`, lalu cari `id` yang sama di log order-service dan product-service, serta baris "email terkirim" dari notification-service.

## Langkah 05: latihan kegagalan

1. `docker compose stop product-service`, lalu `node uji-kegagalan.js` (POST order harus **503**). Jalankan `docker compose ps` dan catat status order-service dan gateway.
2. `docker compose start product-service`, tunggu `healthy` di `docker compose ps`, lalu `node uji-kegagalan.js` (harus **201**).
3. Ubah teks pesan error di order-service, jalankan `docker compose up -d` (tanpa `--build`): perubahan tidak muncul. Lalu `docker compose up -d --build order-service`: perubahan muncul. Kembalikan teks aslinya setelah percobaan.
4. `docker compose down` lalu `docker compose up -d`: stok dan pesanan kembali ke data awal (cek `GET /product-service/products`).

## Langkah 06: CI

Commit dan push ke `main`; buka tab Actions dan pastikan run hijau. Jika merah, lihat log langkah "Build dan nyalakan stack" dan "Tampilkan log".

## Daftar screenshot untuk laporan

1. `docker compose ps` dengan semua service sehat (toko).
2. `npm test` lulus terhadap stack Compose (11 test, fail 0).
3. Latihan kegagalan: 503 saat product-service dihentikan (beserta `docker compose ps` saat itu), lalu 201 setelah dinyalakan kembali.
4. Run GitHub Actions yang hijau pada `main` (tautan run dan tampilan langkah-langkahnya).
5. Tautan repository.
