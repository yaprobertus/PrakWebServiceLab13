![CI](https://github.com/USERNAME/REPO/actions/workflows/ci.yml/badge.svg)

# Praktikum Web Service, Lab 09 sampai Lab 13

Repository ini berisi sistem **Toko** yang dipecah menjadi microservices, dokumentasi API otomatis (OpenAPI/Swagger) pada Lab 11, serta pengujian otomatis, health check, metrik, log berID, monitor, dan CI GitHub Actions pada Lab 12. Lab 13 mengemas seluruh sistem ke dalam container dan menjalankannya dengan Docker Compose (juga di CI).

## Deskripsi singkat sistem

Toko sederhana (produk dan pesanan) dipecah menjadi dua service dengan data masing-masing, dihubungkan lewat HTTP, dan dipasang di balik satu API gateway:

- **product-service** (port 3000): pemilik data produk dan stok. Endpoint: `GET /products`, `GET /products/:id`, `POST /products/:id/reserve`.
- **order-service** (port 3001): pemilik data pesanan. Endpoint: `POST /orders` (memanggil product-service untuk mengurangi stok) dan `GET /orders`. Sejak Lab 10 juga menerbitkan event `order.created` ke RabbitMQ (opsional; jika RabbitMQ tidak jalan, hanya muncul peringatan di log).
- **gateway** (port 8080): satu-satunya alamat untuk klien. Rate limit 20 request per 10 detik, API key (`X-API-Key`) untuk `/order-service`, routing `/product-service/*` dan `/order-service/*`.
- **notification-service** (Lab 10): consumer RabbitMQ, tidak diperlukan untuk Lab 11.

Lab 11 menambahkan dokumentasi code-first: tiap service menyajikan `/openapi.json` dan `/docs` sendiri, dan gateway menggabungkannya menjadi satu halaman.

## Cara menjalankan

Butuh Node.js 18 atau lebih baru (`node -v`). Jalankan dari folder `toko`:

```bash
npm run install:all          # sekali saja
```

Lalu buka tiga terminal, dengan urutan ini:

| Urutan | Terminal | Perintah | Port |
|---|---|---|---|
| 1 | T1 | `cd product-service && node product-service.js` | 3000 |
| 2 | T2 | `cd order-service && node order-service.js` | 3001 |
| 3 | T3 | `cd gateway && node gateway.js` | 8080 |

Cara singkat: `npm run start:all` menjalankan ketiganya sekaligus (Ctrl+C mematikan semuanya). `gateway/.env` dibuat otomatis dari `gateway/.env.example` bila belum ada. Jangan jalankan monolit bersamaan dengan product-service (port 3000 sama).

## Alamat dokumentasi

| Alamat | Isi |
|---|---|
| http://localhost:8080/docs | **Dokumentasi gabungan** (Swagger UI) untuk product-service dan order-service lewat gateway |
| http://localhost:8080/openapi.json | Spesifikasi OpenAPI gabungan (dibuat saat diminta) |
| http://localhost:3000/docs dan /openapi.json | Dokumentasi product-service sendiri |
| http://localhost:3001/docs dan /openapi.json | Dokumentasi order-service sendiri |

Salinan spesifikasi gabungan disimpan di `toko/openapi.json` (hasil `GET :8080/openapi.json`). Untuk mencoba `POST /order-service/orders` dari Swagger UI, klik **Authorize** lalu isi API key yang sama dengan `API_KEY` di `toko/gateway/.env`.

## Variabel environment

| Variabel | Dipakai oleh | Default | Keterangan |
|---|---|---|---|
| `PORT` | semua service | 3000 / 3001 / 8080 | Port masing-masing service |
| `PRODUCT_SERVICE_URL` | gateway, order-service | `http://localhost:3000` | Alamat product-service |
| `ORDER_SERVICE_URL` | gateway | `http://localhost:3001` | Alamat order-service |
| `API_KEY` | gateway | (tanpa default, isi di `gateway/.env`) | API key untuk route `/order-service`; jangan di-commit |
| `PRODUCT_DOC_URL` | gateway | `http://localhost:3000/openapi.json` | Sumber spesifikasi product-service untuk `/docs` gabungan |
| `ORDER_DOC_URL` | gateway | `http://localhost:3001/openapi.json` | Sumber spesifikasi order-service untuk `/docs` gabungan |
| `AMQP_URL` | order-service, notification-service | `amqp://localhost` | Alamat RabbitMQ (Lab 10), jangan menyimpan kredensial di repository |

## Lab 12: pengujian otomatis dan monitoring

Dari folder `toko` (tiga service harus hidup lebih dulu, `npm run start:all`):

```bash
npm test                  # 11 test otomatis lewat gateway (node --test, tanpa paket tambahan)
node monitor.js           # pantau /health gateway, product-service, order-service tiap 5 detik (ALERT / RECOVERED)
node lacak-order.js       # kirim satu order, tampilkan X-Request-Id untuk dicari di log tiga service
```

- Tiap service punya `/health` dan `/metrics` serta menulis satu baris log JSON per permintaan (`observability.js`). Gateway membuat `X-Request-Id`, order-service meneruskannya ke product-service, sehingga satu transaksi terlacak lintas tiga service.
- `proyek-kelompok/` punya `npm test` dan `monitor.js` sendiri (jadwal-service, booking-service, gateway).
- `.github/workflows/ci.yml` menjalankan semua service dan test pada setiap push (dua job: `toko` dan `proyek-kelompok`; sejak Lab 13 lewat Docker Compose).
- Langkah, bukti, dan daftar screenshot: `toko/PANDUAN-LAB12.md`. Refleksi: `toko/docs/refleksi-lab12.md`.

Tanpa nilai `API_KEY` di environment, test dan skrip membaca key dari `toko/gateway/.env`. Di CI, `API_KEY` diisi pada `ci.yml`.

## Menjalankan dengan Docker Compose (Lab 13)

Seluruh sistem toko (gateway, product-service, order-service, notification-service, dan RabbitMQ) dijalankan dengan **satu perintah**, tanpa membuka banyak terminal. Compose file ada di `toko/docker-compose.yml`.

**Prasyarat:** Docker Desktop (Compose v2) sudah berjalan, Node.js 18+ (hanya untuk `npm test`), dan tidak ada service manual atau container `rabbit` dari Lab 10 yang masih berjalan (`docker rm -f rabbit`).

```bash
cd toko
docker compose up --build        # build pertama butuh beberapa menit; Ctrl+C untuk berhenti
```

Di terminal lain (tetap di folder `toko`):

```bash
docker compose ps                # semua service harus Up, yang punya healthcheck "healthy"
npm test                         # 11 test Lab 12 terhadap gateway :8080 (tanpa mengubah test)
docker compose logs order-service product-service notification-service
docker compose down              # matikan dan hapus container (data di memori ikut hilang)
```

| Alamat | Isi |
|---|---|
| http://localhost:8080 | Gateway (satu-satunya pintu masuk klien) |
| http://localhost:8080/docs | Dokumentasi API gabungan (Swagger UI) |
| http://localhost:15672 | Dashboard RabbitMQ |

Port 3000 dan 3001 sengaja **tidak** dibuka ke host. Antar service saling menemukan lewat nama di jaringan Compose (`http://product-service:3000`, `http://order-service:3001`, `amqp://rabbitmq`).

**API key.** Nilai `API_KEY` tidak ditulis di `docker-compose.yml`. Gateway membacanya dari `toko/gateway/.env` (salin dari `.env.example`, atau dibuat otomatis oleh `npm run start:all`; di CI dibuat oleh langkah "Siapkan API key"). `npm test` membaca key dari berkas yang sama, sehingga keduanya cocok. Jangan mengisi `API_KEY` di shell saat memakai Compose. Setelah mengubah kode, bangun ulang dengan `docker compose up -d --build <nama-service>` karena container berisi salinan kode saat build.

Untuk proyek kelompok (`proyek-kelompok/`): `cd proyek-kelompok && docker compose up --build`, lalu `npm test` (gateway di :8080).
Langkah, latihan kegagalan, dan daftar screenshot ada di `toko/PANDUAN-LAB13.md`. Refleksi: `toko/docs/refleksi-lab13.md`.

## Isi repository

- `toko/` : monolit, product-service, order-service, gateway, notification-service, Postman, diagram, panduan Lab 09 sampai 12 (`toko/PANDUAN-LAB12.md`), `toko/tests/`, `toko/monitor.js`, serta `toko/openapi.json`.
- `.github/workflows/ci.yml` : workflow CI (Lab 12, sejak Lab 13 memakai Docker Compose: `docker compose up -d --build --wait`, `npm test`, `docker compose logs`, `docker compose down -v`).
- `proyek-kelompok/` : sistem reservasi tiket kereta dari Lab 08 (Lab 12: `/health`, `/metrics`, log berID, `tests/`, `monitor.js`). Dokumentasi OpenAPI endpoint-nya ada di `proyek-kelompok/docs/openapi-jadwal-service.yaml` dan `proyek-kelompok/docs/openapi-booking-service.yaml` (bahan dokumen desain UAS).

Lab 13 menambahkan `Dockerfile` dan `.dockerignore` di tiap folder service, serta `docker-compose.yml` di `toko/` dan `proyek-kelompok/`.

Catatan: `node_modules` dan `.env` tidak ikut di-commit (sudah diabaikan oleh `.gitignore`).
