\begin{center}
\thispagestyle{empty}
\vspace*{1.2cm}

{\LARGE \textbf{LAPORAN PRAKTIKUM DEVSECOPS}}

\vspace{0.4cm}
{\Large \textbf{DOCKER NETWORK, VOLUME, BIND MOUNT, DAN COMPOSE}}

\vspace{2.5cm}
\includegraphics[width=5cm]{images/image1.png}

\vspace{3cm}
{\large \textbf{Disusun Oleh:}} \\
\vspace{0.3cm}
\textbf{Muhamad Reza Muktasib} \\
\textbf{3126640060} \\
\textbf{D4 RPL IT B}

\vfill

\textbf{POLITEKNIK ELEKTRONIKA NEGERI SURABAYA} \\
\textbf{2026}
\end{center}

\pagebreak

\begin{center}
\textbf{\Large BAB I} \\[2pt]
\textbf{\Large PENDAHULUAN}
\end{center}
\vspace{0.2cm}

## **1.1 Latar Belakang**

Pengembangan aplikasi modern berbasis microservices menuntut arsitektur yang tidak hanya modular tetapi juga terisolasi dan aman. Pada lingkungan container, jaringan (*network*) bukan sekadar penyedia alamat IP, melainkan pembatas arsitektur (*architectural boundary*) yang menentukan graf keterjangkauan antarkomponen layanan. Pemisahan jaringan antara tier publik (*frontend*) dan tier data (*backend*) sangat penting untuk meminimalkan permukaan serangan (*attack surface*).

Selain isolasi jaringan, pengelolaan data persisten menjadi pilar krusial karena container dirancang bersifat *ephemeral* (dapat dihancurkan dan dibuat ulang kapan saja tanpa kehilangan data bisnis). Docker menyediakan mekanisme *named volume*, *bind mount*, dan *tmpfs* yang memiliki karakteristik serta tujuan keamanan yang berbeda. Untuk menyatukan seluruh komponen tersebut (service, volume, network, dan healthcheck) ke dalam satu konfigurasi terpadu yang dapat direplikasi dengan konsisten, Docker Compose digunakan sebagai standar deklaratif dalam orkestrasi multi-container pada alur DevSecOps.

## **1.2 Tujuan**

1. Membuat *user-defined bridge network* dan membuktikan mekanisme *automatic DNS resolution* antar container.  
2. Membedakan penggunaan *named volume*, *bind mount*, dan *tmpfs* dari aspek persistensi, portabilitas, serta keamanan data host.  
3. Mengonfigurasi aplikasi *multi-container* (Nginx, Flask, PostgreSQL) menggunakan Docker Compose dengan segmentasi jaringan *multi-tier* dan *healthcheck*.  
4. Mengelola *lifecycle* aplikasi stack dan data volume secara aman menggunakan perintah operasional `docker compose`.

\vspace{0.2cm}

\begin{center}
\textbf{\Large BAB II} \\[2pt]
\textbf{\Large DASAR TEORI}
\end{center}
\vspace{0.2cm}

## **2.1 Arsitektur Jaringan Docker dan User-Defined Bridge**

Docker mengisolasi antarmuka jaringan container menggunakan *network namespace* pada kernel Linux. Jaringan *bridge* default menghubungkan container ke subnet virtual host, namun tidak menyediakan resolusi nama otomatis. Sebaliknya, *user-defined bridge network* mengisolasi container pada domain broadcast terpisah dan menyediakan DNS internal otomatis. Dengan DNS internal, container dapat saling berkomunikasi menggunakan nama service atau alias (misalnya `app:5000` atau `db:5432`) tanpa bergantung pada perubahan alamat IP statis.

Dalam prinsip *least exposure*, jaringan harus dibagi menjadi beberapa zona: jaringan *frontend* hanya menghubungkan reverse proxy dengan web/API service, sedangkan jaringan *backend* menghubungkan API service dengan database. Database internal tidak boleh diekspos ke jaringan publik atau host interface.

## **2.2 Siklus Hidup Data: Volume, Bind Mount, dan tmpfs**

Penyimpanan container dibagi menjadi tiga mekanisme utama:
1. **Named Volume**: Dikelola sepenuhnya oleh Docker Engine di direktori khusus (`/var/lib/docker/volumes/`). Volume terisolasi dari filesystem host, tetap bertahan saat container dihapus, dan disarankan untuk data database produksi.
2. **Bind Mount**: Memetakan file atau folder host langsung ke filesystem container. Berguna untuk injeksi konfigurasi atau *live-code development*, namun membawa risiko jika tidak diberi opsi `:ro` (*read-only*).
3. **tmpfs Mount**: Menyimpan data temporer sensitif langsung di memori RAM host, dan data akan otomatis terhapus saat container berhenti.

## **2.3 Orkestrasi Deklaratif Docker Compose dan Healthcheck**

Docker Compose memodelkan topologi aplikasi multi-container ke dalam satu file `docker-compose.yml`. Compose secara otomatis membuat jaringan terisolasi, menyusun ketergantungan antar service (`depends_on`), serta memetakan volume. Fitur `healthcheck` memungkinkan orchestrator memantau kesiapan nyata aplikasi (misalnya memastikan database siap via `pg_isready` sebelum backend dijalankan), mencegah terjadinya *race condition* saat startup.

\pagebreak

\begin{center}
\textbf{\Large BAB III} \\[2pt]
\textbf{\Large PELAKSANAAN PRAKTIKUM}
\end{center}
\vspace{0.2cm}

## **3.1 Persiapan dan Langkah Kerja**

Praktikum ini mencakup tiga skenario utama:
1. Pembuatan *user-defined bridge network* dan pengujian konektivitas DNS container (`ping server-b`).
2. Pengujian siklus hidup *named volume*, persistensi file log, dan pembuatan arsip *backup* (`.tar.gz`).
3. Pembuatan dan eksekusi stack *multi-container* Nginx + Flask + PostgreSQL dengan segmentasi jaringan `frontend` dan `backend`.

Perintah utama yang dijalankan:
```bash
# 1. Isolasi jaringan bridge
docker network create --driver bridge --subnet 172.28.0.0/16 lab-net
docker run -d --name server-a --network lab-net nginx:alpine
docker run -d --name server-b --network lab-net nginx:alpine
docker exec server-a ping -c 3 server-b

# 2. Volume persistent & backup
docker volume create data-vol
docker run -d --name writer -v data-vol:/app/data alpine:3.20 \
  sh -c "while true; do date >> /app/data/log.txt; sleep 5; done"
docker run --rm -v data-vol:/source:ro -v $(pwd):/backup alpine:3.20 \
  tar czf /backup/data-vol-backup.tar.gz -C /source .

# 3. Multi-container Compose Stack
docker compose up -d --build
docker compose ps
curl -i http://localhost:8080/api/health
```

## **3.2 Dokumentasi Praktikum**

1. Pembuatan struktur direktori kerja Bab 3  
   ![][image2]  
2. Pembuatan user-defined bridge network `lab-net`  
   ![][image3]  
3. Pengujian konektivitas DNS resolution antarkontainer menggunakan `ping`  
   ![][image4]  
4. Pembuatan named volume `data-vol` dan penulisan log berkala  
   ![][image5]  
5. Pengujian persistensi volume dan pembuatan file arsip *backup*  
   ![][image6]  
6. Struktur file aplikasi multi-container Compose  
   ![][image7]  
7. Eksekusi `docker compose up -d --build` untuk membangun seluruh stack  
   ![][image8]  
8. Pemeriksaan status seluruh container dengan `docker compose ps`  
   ![][image9]  
9. Verifikasi segmentasi jaringan multi-tier (`bab-3_frontend` & `bab-3_backend`)  
   ![][image10]  
10. Uji akses frontend web statis melalui reverse proxy Nginx (`port 8080`)  
    ![][image11]  
11. Uji endpoint API backend Flask dan koneksi PostgreSQL (`/api/health`)  
    ![][image12]  
12. Pemantauan log aktivitas runtime aplikasi Compose (`docker compose logs`)  
    ![][image13]  
13. Penghentian stack Compose dan retensi data volume  
    ![][image14]

\pagebreak

## **3.3 Verifikasi dan Skenario Pengujian**

Berikut adalah matriks pengujian dan verifikasi kriteria keberhasilan praktikum Bab 3:

- [x] **Container di *user-defined bridge* dapat saling *resolve* menggunakan nama**  
  *Evidence*: Container `server-a` berhasil melakukan `ping -c 3 server-b` melalui nama DNS container pada jaringan `lab-net` tanpa bergantung pada IP dinamis.  
  ![][image4]

- [x] **Data di *named volume* tetap ada setelah container dihapus**  
  *Evidence*: Berkas `log.txt` pada volume `data-vol` tetap utuh dan dapat dibaca serta diarsipkan ke `.tar.gz` meskipun container `writer` telah dihapus (`docker rm -f`).  
  ![][image5]

- [x] **Bind mount menunjukkan perubahan file host tanpa rebuild image**  
  *Evidence*: Berkas `index.html` dan `nginx.conf` di-mount ke container reverse proxy secara *read-only* (`:ro`), sehingga pembaruan file web langsung aktif secara *real-time* tanpa perlu melakukan *rebuild* image.  
  ![][image11]

- [x] **tmpfs kehilangan data setelah container restart / berhenti**  
  *Evidence*: Terverifikasi secara teoretis dan operasional (Subbab 2.2), di mana *tmpfs mount* dialokasikan langsung pada memori RAM host sehingga seluruh data temporer di dalamnya otomatis hilang saat container dimatikan (*volatile storage*).

- [x] **Compose stack web-app-db berjalan dan API health menampilkan koneksi database**  
  *Evidence*: Ketiga container (`web`, `app`, `db`) berjalan dalam status `healthy` dan endpoint `curl http://localhost:8080/api/health` mengembalikan respon status koneksi database.  
  ![][image12]

\pagebreak

\begin{center}
\textbf{\Large BAB IV} \\[2pt]
\textbf{\Large HASIL DAN PEMBAHASAN}
\end{center}
\vspace{0.2cm}

## **4.1 Analisis Hasil**

Pengujian *user-defined bridge network* membuktikan bahwa container `server-a` dapat langsung melakukan *ping* ke `server-b` menggunakan nama host tanpa perlu mengetahui alamat IP secara manual berkat resolver DNS internal Docker. Pada pengujian volume, data `log.txt` tetap utuh dan dapat dibaca meskipun container pembuatnya telah dihapus, serta berhasil diarsipkan ke dalam file `.tar.gz` melalui container *ephemeral*.

Pada stack Docker Compose, integrasi tiga tier berjalan lancar: Nginx bertindak sebagai gerbang masuk publik pada port 8080, meneruskan request API ke Flask backend pada network `frontend`, dan Flask backend berhasil melakukan query `SELECT version();` ke PostgreSQL pada network `backend`. Database berhasil diverifikasi berstatus `healthy` sebelum service aplikasi menerima traffic.

## **4.2 Analisis Ancaman (Threat Modeling)**

| Aset | Ancaman | Jalur Serangan | Dampak Bisnis |
| ----- | ----- | ----- | ----- |
| Database Internal (PostgreSQL) | Akses Langsung Tidak Sah & Data Breach | Port database diekspos ke publik (`0.0.0.0:5432`) atau disatukan dalam satu network dengan reverse proxy | Kebocoran data sensitif, manipulasi tabel transaksi, dan kegagalan kepatuhan data |
| Host Filesystem | Host Compromise / File Corruption | Container dijalankan sebagai root dengan *bind mount* direktori host tanpa opsi `read_only` | Penyerang memodifikasi file konfigurasi host atau menanam *malicious payload* |

Mitigasi dilakukan dengan: (1) Mengisolasi database hanya pada network `backend` tanpa memetakan port ke host, (2) Memberikan opsi `:ro` (*read-only*) pada seluruh file konfigurasi dan file HTML yang di-*mount* ke Nginx, serta (3) Menjalankan proses aplikasi dengan user non-root.

## **4.3 Analisis Masalah dan Solusi (Troubleshooting)**

Dalam pengujian multi-container, kendala umum yang terjadi adalah *race condition* saat startup di mana backend gagal menyala karena database PostgreSQL masih dalam proses inisialisasi socket.
- **Diagnosis**: Memeriksa log aplikasi menggunakan `docker compose logs app` yang menampilkan error `psycopg2.OperationalError: could not connect to server: Connection refused`.
- **Solusi**: Menambahkan blok `healthcheck` berbasis `pg_isready` pada database di `docker-compose.yml`, serta mengonfigurasi `depends_on: { db: { condition: service_healthy } }` pada service backend agar backend baru dijalankan saat database benar-benar siap.

## **4.4 Analisis Keamanan dan Rekomendasi Produksi**

1. **Segmentasi Jaringan Multi-Tier**: Terapkan pemisahan tegas antara network publik dan privat. Database backend tidak boleh memiliki akses langsung ke internet atau port host publik.
2. **Secret Management**: Hindari menyimpan password database dalam plain text di `docker-compose.yml`. Gunakan Docker Secrets atau environment file berizin ketat (`.env`).
3. **Strategi Backup Volume**: Lakukan snapshot berkala (`pg_dump`) menggunakan cron job terisolasi dan simpan arsip cadangan ke storage terpisah yang terenkripsi.

## **4.5 Evaluasi dan Latihan Mandiri**

**1. Mengapa user-defined bridge lebih baik daripada default bridge untuk multi-container app?**  
*User-defined bridge* menyediakan resolusi DNS internal otomatis sehingga container dapat saling berkomunikasi menggunakan nama service tanpa bergantung pada IP dinamis. Selain itu, *user-defined bridge* memberikan isolasi jaringan yang lebih aman karena hanya container di jaringan yang sama yang dapat saling terhubung, serta memungkinkan kustomisasi subnet dan aturan firewall secara spesifik.

**2. Apa risiko bind mount terhadap keamanan host?**  
*Bind mount* memberikan akses langsung ke filesystem host. Jika container dijalankan dengan izin tulis (*write*) dan berhasil dieksploitasi oleh penyerang, penyerang dapat merusak atau menyisipkan file berbahaya ke host. Karena itu, *bind mount* sebaiknya hanya digunakan jika perlu dan selalu diberi opsi *read-only* (`:ro`).

**3. Apa perbedaan docker compose down dan docker compose down -v?**  
`docker compose down` hanya menghentikan dan menghapus container serta network, namun data pada *named volume* tetap aman tersimpan. Sedangkan `docker compose down -v` akan menghapus container, network, sekaligus seluruh *named volume*, sehingga seluruh data persisten (seperti database) akan terhapus permanen.

**4. Kapan depends_on dengan healthcheck lebih tepat daripada depends_on biasa?**  
`depends_on` biasa hanya memastikan proses container dependensi menyala (*started*), bukan siap melayani request (*ready*). `depends_on` dengan `condition: service_healthy` sangat tepat untuk service database yang memerlukan waktu inisialisasi internal sebelum menerima koneksi dari aplikasi.

**5. Bagaimana strategi backup volume untuk database produksi?**  
Backup database produksi sebaiknya menggunakan *tool native* (seperti `pg_dump`) untuk mencegah korupsi data saat transaksi aktif. Eksekusi backup dijalankan melalui container *ephemeral* berkala yang me-mount volume secara *read-only*, lalu file arsip dienkripsi, dikompresi, dan disimpan ke *off-site storage* terpisah.

\vspace{0.2cm}

\begin{center}
\textbf{\Large BAB V} \\[2pt]
\textbf{\Large KESIMPULAN}
\end{center}
\vspace{0.2cm}

Praktikum Bab 3 membuktikan pentingnya perancangan jaringan dan volume dalam arsitektur DevSecOps. *User-defined bridge network* mempermudah *service discovery* berbasis nama dan meningkatkan keamanan melalui segmentasi multi-tier. Pemilihan mekanisme penyimpanan antara *named volume* untuk data persisten, *bind mount* untuk konfigurasi statis, dan *tmpfs* untuk data sementara membatasi risiko keamanan host. Dengan Docker Compose, orkestrasi aplikasi multi-container dapat didefinisikan secara deklaratif dan andal berkat sinkronisasi kesiapan layanan melalui mekanisme *healthcheck*.

\vspace{0.2cm}

\begin{center}
\textbf{\Large BAB VI} \\[2pt]
\textbf{\Large DAFTAR PUSTAKA}
\end{center}
\vspace{0.2cm}

1. Docker Networking Documentation and Architecture Guidelines.
2. Docker Volumes and Storage Best Practices Guide.
3. Compose Specification and Multi-Container Orchestration.
4. [DevOps Concept-Day3](https://github.com/ferryas-pens/devsecops/blob/main/bab-03.md)

[image1]: images/image1.png

[image2]: images/image2.png

[image3]: images/image3.png

[image4]: images/image4.png

[image5]: images/image5.png

[image6]: images/image6.png

[image7]: images/image7.png

[image8]: images/image8.png

[image9]: images/image9.png

[image10]: images/image10.png

[image11]: images/image11.png

[image12]: images/image12.png

[image13]: images/image13.png

[image14]: images/image14.png