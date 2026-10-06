\begin{center}
\thispagestyle{empty}
\vspace*{1.2cm}

{\LARGE \textbf{LAPORAN PRAKTIKUM DEVSECOPS}}

\vspace{0.4cm}
{\Large \textbf{DATABASE SERVICE DI DOCKER: POSTGRESQL, PGADMIN, VOLUME, BACKUP, DAN RESTORE}}

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

Penerapan database di dalam arsitektur kontainer memiliki paradigma yang mendasar dan berbeda dibandingkan dengan layanan tanpa *state* (*stateless services*) seperti *web server* atau *reverse proxy*. Pada aplikasi *stateless*, kontainer dapat di-*destroy* dan di-*recreate* kapan saja tanpa risiko hilangnya data. Sebaliknya, pada layanan *database* (*stateful*), data merupakan aset inti sistem yang harus memiliki jaminan persistensi, integritas, dan ketersediaan tinggi melampaui siklus hidup (*lifecycle*) kontainer itu sendiri.

Melalui praktikum Bab 5 ini, dibangun infrastruktur database relasional modern menggunakan PostgreSQL 16 Alpine dan pgAdmin 4 berbasis Docker Compose. Infrastruktur ini mengintegrasikan mekanisme *named volume* untuk persistensi direktori data `/var/lib/postgresql/data`, inisialisasi skema dan data awal otomatis melalui `/docker-entrypoint-initdb.d`, isolasi *private network* `data-net`, *healthcheck* berkala berbasis utilitas `pg_isready`, serta integrasi antarmuka visual pgAdmin dengan konfigurasi *pre-registered server*. Selain itu, praktikum ini memprioritaskan aspek *Disaster Recovery* melalui otomatisasi *logical backup* dengan utilitas `pg_dump` berformat *custom*, verifikasi integritas berkas menggunakan *cryptographic hash* SHA-256, serta pengujian pemulihan data (*restore testing*) pada database terpisah untuk membuktikan keabsahan *backup* data secara nyata.

## **1.2 Tujuan**

1. Mampu mengonfigurasi dan menjalankan *multi-container orchestration* PostgreSQL 16 dan pgAdmin 4 menggunakan Docker Compose secara terisolasi dan terintegrasi.
2. Mampu merancang skema database relasional serta mengotomatisasi inisialisasi tabel dan data awal memanfaatkan direktori `/docker-entrypoint-initdb.d`.
3. Mampu membuktikan persistensi data melintasi siklus hidup (*lifecycle*) kontainer menggunakan *named volume*.
4. Mampu menyusun skrip otomatisasi *logical backup* berbasis `pg_dump` dengan validasi integritas *hash* SHA-256.
5. Mampu melakukan prosedur *restore testing* pada database terpisah guna menjamin keandalan *disaster recovery*.
6. Mampu menganalisis prinsip *least privilege*, manajemen rahasia (*secret management*), isolasi port, serta tata kelola data operasional.

\pagebreak

\begin{center}
\textbf{\Large BAB II} \\[2pt]
\textbf{\Large DASAR TEORI}
\end{center}

## **2.1 Analisis Arsitektur: Database Container vs Stateless Service & Persistensi**

Kontainerisasi menyatukan *runtime engine* dan dependensi aplikasi ke dalam unit yang portabel dan terisolasi. Ketika diterapkan pada database seperti PostgreSQL, *engine* Docker mengelola proses komputasi, namun lapisan *writable layer* kontainer bersifat fana (*ephemeral*). Setiap perubahan data yang disimpan langsung pada lapisan kontainer akan musnah saat kontainer dihapus. Oleh karena itu, arsitektur database memerlukan pemisahan ketat antara lapisan *compute* (kontainer PostgreSQL) dan lapisan *storage* (Docker Volume).

| Environment Variables | Fungsi Operasional |
| :--- | :--- |
| `POSTGRES_DB` | Nama database awal yang dibuat ketika inisialisasi kluster pertama kali |
| `POSTGRES_USER` | Initial user / superuser pengelola kontainer database |
| `POSTGRES_PASSWORD` | Initial password untuk autentikasi pengguna database |
| `PGDATA` | Path direktori penyimpanan data internal PostgreSQL di dalam kontainer |
| `POSTGRES_INITDB_ARGS` | Argumen tambahan untuk perintah `initdb`, seperti pengaturan *encoding* atau *locale* |

| Tipe Storage | Path Host | Karakteristik Lifecycle | Rekomendasi Penggunaan |
| :--- | :--- | :--- | :--- |
| **Named Volume** | `/var/lib/docker/volumes/` | Dikelola penuh Docker, persisten melintasi *recreate* kontainer | Standar utama persistensi database produksi dan lab |
| **Bind Mount** | Sembarang direktori di Host | Bergantung pada struktur direktori host dan izin berkas (*permissions*) | Inisialisasi konfigurasi, skrip SQL init, atau berkas *backup* |
| **tmpfs Mount** | Memori Utama (RAM) Host | Bersifat volatil, data musnah saat kontainer berhenti (*stop*) | Data sangat sensitif sementara atau *temporary cache* cepat |

Empat pilar utama persistensi data database mencakup:

1. **Durability Transaksi** diwujudkan melalui mekanisme *Write-Ahead Logging* (WAL) internal PostgreSQL yang memastikan setiap transaksi terkomit dicatat secara permanen sebelum dituliskan ke blok tabel.
2. **Keberlangsungan Media** diwujudkan melalui pemasangan *named volume* yang terisolasi dari *writable layer* kontainer.
3. **Kemampuan Backup** diwujudkan melalui ekstraksi data yang konsisten dan portabel menggunakan utilitas `pg_dump`.
4. **Kemampuan Restore** diwujudkan melalui pembuktian nyata pemulihan data pada target database yang teruji secara fungsional.

\pagebreak

## **2.2 Peta Konsep: Manajemen Database Container dan Persistensi Volume**

Penerapan database di Docker memerlukan *orchestration* yang mengintegrasikan *storage*, siklus inisialisasi, *network*, dan pemantauan. Pemetaan konsep disajikan pada *Mind Map* berikut:

\begin{center}
\includegraphics[width=0.98\linewidth]{images/peta_konsep_database.png}
\end{center}

Berdasarkan *Mind Map* di atas, arsitektur pengelolaan database kontainer mencakup empat aspek utama:

1. **Named Volume dan Storage** memisahkan direktori penyimpanan internal `/var/lib/postgresql/data` ke dalam *named volume* `pg-data`. Pengaturan ini menjamin data tetap utuh meskipun kontainer dimatikan, dihapus, atau diperbarui versinya.
2. **Inisialisasi Skema Initdb** memanfaatkan mekanisme bawaan citra resmi PostgreSQL yang secara otomatis mengeksekusi berkas `.sql` atau `.sh` pada direktori `/docker-entrypoint-initdb.d/` hanya saat volume data dalam kondisi kosong.
3. **Isolasi Network dan Port** menempatkan PostgreSQL dan pgAdmin pada *bridge network* privat `data-net`, serta membatasi pemetaan port host ke alamat *loopback* `127.0.0.1` sehingga database terlindung dari akses publik luar.
4. **Healthcheck dan Integrasi pgAdmin** menerapkan pemeriksaan kesiapan berkala melalui utilitas `pg_isready` serta melakukan *pre-registration* server PostgreSQL melalui berkas `servers.json` bermodus *read-only*.

\pagebreak

## **2.3 Peta Konsep: Strategi Backup Logical dan Restore Testing**

Pencadangan data tidak memiliki nilai apabila tidak dapat di-*restore*. Strategi *backup* dan verifikasi *restore* dipetakan pada *Mind Map* berikut:

\begin{center}
\includegraphics[width=0.98\linewidth]{images/peta_konsep_backup.png}
\end{center}

Berdasarkan *Mind Map* di atas, tata kelola perlindungan data dijalankan melalui empat mekanisme:

1. **Logical Backup pg_dump** mengekstrak struktur skema dan data ke dalam format *custom archive* (`-Fc`) yang ringkas, terkompresi, dan portabel lintas arsitektur CPU dengan menghilangkan metadata kepemilikan lokal (`--no-owner --no-privileges`).
2. **Integritas Data dan Proteksi Secret** memvalidasi keberadaan dan ukuran berkas menggunakan perintah pengujian ukuran, membangkitkan *cryptographic hash* SHA-256 untuk mendeteksi *file corruption*, serta mengamankan kredensial dalam berkas `.env` yang diabaikan oleh Git.
3. **Prosedur Restore Testing** menjalankan uji pemulihan berkas *dump* ke database pengujian terpisah (`labdb_restore_test`) menggunakan `pg_restore --clean` tanpa mengganggu ataupun menimpa database utama yang sedang melayani transaksi.
4. **Verifikasi dan Tata Kelola** memvalidasi keutuhan baris data hasil *restore* melalui kueri agregasi, mengevaluasi *Recovery Point Objective* (RPO) dan *Recovery Time Objective* (RTO), serta membersihkan database uji setelah verifikasi selesai.

\pagebreak

\begin{center}
\textbf{\Large BAB III} \\[2pt]
\textbf{\Large METODOLOGI DAN LANGKAH KERJA}
\end{center}

## **3.1 Arsitektur Laboratorium dan Struktur Direktori**

Praktikum disusun dalam satu *workspace* terstruktur `~/docker-lab/bab-5/` untuk memisahkan skrip konfigurasi, inisialisasi skema, dan arsip *backup*:

```text
bab-5/
├── .env.example              # Template konfigurasi environment variables
├── .env                      # Environment variables aktif (permissions ketat 600)
├── .gitignore                # Pengecualian berkas kredensial dan dump dari Git
├── compose.yaml              # Multi-container orchestration PostgreSQL & pgAdmin
├── init/
│   └── 01-schema.sql         # Skrip inisialisasi tabel students dan seed data
├── pgadmin/
│   └── servers.json          # Konfigurasi pre-registrasi koneksi pgAdmin
├── scripts/
│   ├── backup.sh             # Skrip otomatisasi logical dump & SHA-256
│   └── restore-test.sh       # Skrip restore testing ke database terpisah
└── backup/                   # Direktori penampung berkas dump dan checksum
```

## **3.2 File Konfigurasi dan Otomatisasi Skrip**

Infrastruktur praktikum dibangun melalui berkas-berkas konfigurasi berikut:

1. **Berkas Environment Variables (`.env` dan `.gitignore`)** menampung variabel `POSTGRES_DB=labdb`, `POSTGRES_USER=labuser`, `POSTGRES_PASSWORD=labpass123`, serta kredensial akun pgAdmin. Berkas `.gitignore` memastikan `.env` dan direktori `backup/` tidak terunggah ke *version control*.
2. **Berkas Orchestration (`compose.yaml`)** mendefinisikan *service* `postgres-db` (citra `postgres:16-alpine`, *named volume* `pg-data`, *mount* `./init` bermodus `:ro`, serta *healthcheck* `pg_isready`) dan *service* `pgadmin` (citra `dpage/pgadmin4:latest` dengan *dependency condition* `service_healthy`).
3. **Skrip Inisialisasi (`init/01-schema.sql`)** membangun tabel `students` dengan *constraint* `PRIMARY KEY`, `UNIQUE (nrp)`, `NOT NULL`, serta indeks `idx_students_name` dan pengisian 2 baris data awal.
4. **Konfigurasi Server GUI (`pgadmin/servers.json`)** mendaftarkan server database dengan *hostname* internal kontainer `postgres-db`, port `5432`, dan database `labdb`.
5. **Skrip Backup dan Restore Test (`scripts/backup.sh` dan `scripts/restore-test.sh`)** mengotomatisasi eksekusi `pg_dump` format *custom*, kalkulasi *checksum* SHA-256, serta pemulihan ke database `labdb_restore_test`.

## **3.3 Prosedur Eksekusi dan Pengujian**

Tahapan praktikum dijalankan melalui urutan operasional terstandar:

1. Menyiapkan struktur direktori dan menetapkan izin eksekusi (`chmod +x scripts/*.sh`).
2. Melakukan validasi sintaks *orchestration* menggunakan perintah `docker compose config`.
3. Menjalankan *stack* layanan dalam mode latar belakang (`docker compose up -d`) dan memeriksa *health status* kontainer.
4. Melakukan verifikasi inisialisasi skema melalui utilitas interaktif `psql`.
5. Menguji persistensi volume dengan menyisipkan baris data baru, mematikan kontainer (`down`), menyalakan kembali (`up`), dan memverifikasi ketersediaan data.
6. Menjalankan skrip *backup* data, memverifikasi nilai *checksum* SHA-256, dan mengeksekusi uji *restore* ke database terpisah.

\pagebreak

\begin{center}
\textbf{\Large BAB IV} \\[2pt]
\textbf{\Large HASIL DAN PEMBAHASAN}
\end{center}

## **4.1 Validasi Konfigurasi dan Deployment Multi-Container**

Pembangunan lingkungan laboratorium diawali dengan pembuatan struktur direktori terisolasi, pembuatan berkas *environment variables* `.env`, konfigurasi pengabaian Git `.gitignore`, serta penyusunan berkas *orchestration* `compose.yaml`.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image2.png}
\end{center}

Setelah struktur direktori siap, konfigurasi *environment variables* didefinisikan untuk memisahkan kredensial dari berkas *orchestration*, disertai pembuatan berkas `.gitignore` untuk mencegah kebocoran data sensitif dan berkas *backup* ke repositori publik.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image3.png}
\end{center}

\pagebreak

Seluruh dependensi layanan, volume persisten, aturan jaringan terisolasi, serta mekanisme *healthcheck* diintegrasikan ke dalam berkas `compose.yaml`.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image4.png}
\end{center}

Hak akses eksekusi kemudian diberikan kepada seluruh skrip otomatisasi, dilanjutkan dengan validasi sintaks *orchestration* menggunakan perintah `docker compose config` guna memastikan tidak ada kesalahan format YAML sebelum kontainer dijalankan.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image9.png}
\end{center}

\pagebreak

Setelah konfigurasi dinyatakan valid, *stack multi-container* diluncurkan ke latar belakang menggunakan perintah `docker compose up -d`. Sistem secara otomatis membuat jaringan `data-net`, menginisialisasi volume `pg-data` dan `pgadmin-data`, serta menjalankan kontainer database dan antarmuka web.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image10.png}
\end{center}

Pemantauan status kontainer melalui `docker compose ps` menunjukkan bahwa kontainer `postgres-db` berhasil mencapai status `healthy` setelah melewati pemeriksaan kesiapan `pg_isready`, sehingga kontainer `pgadmin` dapat berjalan normal tanpa mengalami kegagalan koneksi.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image11.png}
\end{center}

\pagebreak

## **4.2 Inisialisasi Skema Otomatis dan Akses CLI psql**

Inisialisasi tabel dan data awal dikelola melalui berkas `init/01-schema.sql` yang dimuat secara *read-only* ke dalam direktori `/docker-entrypoint-initdb.d/`.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image5.png}
\end{center}

Pemeriksaan log kontainer `postgres-db` membuktikan bahwa *entrypoint script* mendeteksi ketiadaan data pada volume baru, lalu mengeksekusi skrip `01-schema.sql` secara berurutan hingga sistem siap menerima koneksi client.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image12.png}
\end{center}

\pagebreak

Verifikasi struktur tabel dan data awal dilakukan melalui CLI `psql`. Perintah `\dt` mengonfirmasi keberadaan relasi tabel `students`, dan kueri `SELECT` menampilkan dua baris data mahasiswa yang terinisialisasi secara otomatis.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image13.png}
\end{center}

## **4.3 Pengujian Persistensi Volume Data**

Pengujian persistensi dilakukan dengan menyisipkan baris data ketiga (`31230003`, `Mahasiswa Tiga`) ke dalam database utama melalui perintah `INSERT`.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image14.png}
\end{center}

\pagebreak

Untuk membuktikan bahwa data tidak terikat pada siklus hidup kontainer, seluruh kontainer dihentikan dan dihapus menggunakan perintah `docker compose down`, kemudian dinyalakan kembali dengan `docker compose up -d`. Hasil kueri membuktikan ketiga baris data tetap utuh tersimpan di dalam *named volume* `pg-data`.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image15.png}
\end{center}

## **4.4 Otomatisasi Backup Logical dan Validasi Checksum SHA-256**

Pencadangan data logis diotomatisasi melalui skrip `scripts/backup.sh` yang memanfaatkan utilitas `pg_dump` dengan format *custom* (`-Fc`) dan membangkitkan berkas *checksum* SHA-256.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image7.png}
\end{center}

\pagebreak

Eksekusi skrip pencadangan menghasilkan berkas arsip `labdb-*.dump` dan berkas integritas `.sha256`. Verifikasi menggunakan perintah `sha256sum --check` memberikan status `OK`, membuktikan berkas *backup* tidak mengalami *file corruption*.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image16.png}
\end{center}

## **4.5 Pengujian Restore Data ke Database Terpisah**

Pengujian pemulihan diatur oleh skrip `scripts/restore-test.sh` yang melakukan validasi *hash*, menghapus database uji lama jika ada, membuat database `labdb_restore_test`, lalu memulihkan data menggunakan `pg_restore`.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image8.png}
\end{center}

\pagebreak

Eksekusi skrip pemulihan membuktikan keberhasilan *Disaster Recovery*, di mana kueri agregasi `SELECT COUNT(*)` menghasilkan nilai 3 yang sesuai dengan kondisi database saat dicadangkan.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image17.png}
\end{center}

Pemeriksaan rinci terhadap tabel `students` pada database `labdb_restore_test` mengonfirmasi bahwa seluruh atribut record mahasiswa berhasil dipulihkan secara sempurna tanpa kehilangan integritas.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image18.png}
\end{center}

\pagebreak

## **4.6 Monitoring Kesiapan Database dan Manajemen Resource**

Pemantauan status operasional dilakukan melalui utilitas `pg_isready` untuk memastikan soket PostgreSQL menerima koneksi, didukung pemeriksaan alokasi penyimpanan volume fisik melalui `docker volume ls` dan `docker system df -v`.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image19.png}
\end{center}

Integrasi antarmuka web pgAdmin 4 didukung oleh berkas konfigurasi `pgadmin/servers.json` yang secara otomatis mendaftarkan server `postgres-db` ke dalam panel navigasi pengguna.

\begin{center}
\includegraphics[width=0.75\linewidth]{images/image6.png}
\end{center}

\pagebreak

## **4.7 Evaluasi dan Latihan Mandiri**

**1. Mengapa init script tidak dijalankan ulang saat volume lama masih ada?**  
Karena dari arsitektur Docker image PostgreSQL resminya memang sengaja dibuat begitu untuk melindungi data yang sudah ada. Skrip SQL atau shell yang ditaruh di `/docker-entrypoint-initdb.d` cuma bakal dieksekusi sekali saja saat direktori data `/var/lib/postgresql/data` masih benar-benar kosong (proses *initdb* awal). Kalau volume `pg-data` sudah terisi kluster database lama, script *entrypoint* kontainer otomatis langsung melewati langkah init dan langsung menjalankan service database secara normal. Tujuannya supaya data kita yang sudah berjalan tidak tertimpa atau error karena *duplicate table*. Jadi kalau mau ubah skema pas database sudah ada isinya, cara yang benar adalah lewat skrip migrasi SQL manual, bukan berharap init script-nya jalan lagi.

**2. Apa risiko menaruh password database pada docker-compose.yml?**  
Risiko utamanya adalah kredensial penting kita sangat gampang bocor. Berkas `compose.yaml` biasanya kita simpan dan kita *push* ke repositori Git/GitHub, sehingga siapa saja yang punya akses baca ke repo bisa langsung melihat username dan password plaintext tersebut. Selain itu, siapa pun yang punya akses shell di server bisa membaca kata sandinya dengan mudah lewat perintah `docker inspect <container>`. Makanya praktik amannya adalah memisahkan nilai rahasia ini ke berkas `.env` lokal, mengatur *permissions* berkasnya jadi `600`, dan memastikan `.env` sudah masuk ke `.gitignore` agar tidak ikut terunggah.

**3. Bagaimana cara membuktikan backup dapat dipulihkan?**  
Tidak cukup hanya melihat ada berkas `.dump` yang ukurannya tidak nol di folder penyimpanan. Cara membuktikan yang sebenarnya adalah dengan melakukan uji coba *restore* secara nyata ke sebuah database baru/terpisah (seperti `labdb_restore_test` pada lab ini). Setelah proses `pg_restore` selesai, kita wajib menjalankan kueri verifikasi seperti `SELECT COUNT(*)` dan mengecek integritas isi datanya. Kalau jumlah baris dan seluruh record-nya sama persis dengan database sumber tanpa ada pesan error, barulah kita punya bukti valid bahwa cadangan data tersebut benar-benar bisa dipulihkan saat terjadi insiden *disaster recovery*.

**4. Apa bedanya logical backup pg_dump dan backup filesystem volume mentah?**  
Bedanya terletak pada level data yang dicadangkan. `pg_dump` merupakan *logical backup*, artinya dia mengekstrak perintah DDL skema dan baris data tabel menjadi berkas arsip terstruktur. Keunggulannya adalah sangat fleksibel dan portabel, berkas dump bisa kita *restore* ke server lain dengan arsitektur CPU berbeda atau versi PostgreSQL yang sedikit berbeda. Sedangkan backup filesystem mentah adalah meng-copy langsung blok folder fisik `/var/lib/postgresql/data`. Backup fisik ini rawan korup kalau disalin saat database masih aktif melayani transaksi penulisan (tanpa *checkpoint* yang benar), dan sangat kaku karena terikat ketat pada versi biner dan filesystem spesifik mesin tersebut.

**5. Apa dampak docker compose down -v terhadap database?**  
Perintah `docker compose down -v` akan menghapus kontainer, jaringan, dan sekaligus menghapus seluruh *named volume* yang terdaftar (yaitu `pg-data` dan `pgadmin-data`) dari storage Docker host secara permanen. Dampaknya, semua tabel, record mahasiswa yang sudah kita inputkan, dan konfigurasi database akan musnah total dan tidak bisa dikembalikan lagi kecuali kita punya berkas cadangan *dump*. Ini berbeda dengan perintah `docker compose down` biasa tanpa flag `-v`, yang hanya mematikan kontainernya saja tetapi tetap membiarkan data di dalam volume tetap utuh dan aman.

\pagebreak

**6. Bagaimana cara terbaik mengelola dan menyimpan credentials database di lingkungan production?**  
Kalau di lingkungan *production*, kita sudah tidak boleh lagi menyimpan password di berkas `.env` biasa di server apalagi sampai di-*hardcode* ke berkas konfigurasi. Praktik terbaik yang biasa saya terapkan mencakup beberapa lapisan:

1. **GitHub Actions Secrets pada CI/CD** mengamankan seluruh nilai sensitif (seperti kredensial koneksi DB, token deployment, atau master key) pada fitur rahasia repositori (*Repository Secrets*). Nilai ini dienkripsi dan di-*inject* secara dinamis hanya saat *job deployment* berjalan tanpa pernah terekspos di log console pipeline.
2. **Dedicated Secrets Manager** mengamankan kredensial untuk penyimpanan terpusat tingkat *enterprise* pada *cloud secret manager* (seperti HashiCorp Vault, AWS Secrets Manager, atau GCP Secret Manager). Layanan ini menyediakan kontrol hak akses tersentralisasi (*least privilege*), pencatatan audit akses (*access logs*), serta kemampuan rotasi kata sandi otomatis secara berkala (*automatic credential rotation*).
3. **Mekanisme Secret Injection di Kubernetes** menyuntikkan kredensial secara dinamis saat *pod* dibuat (*on-deploy injection*) menggunakan Kubernetes Secrets atau operator khusus seperti *Vault Agent Injector* dan *External Secrets Operator*. Mekanisme ini menyuntikkan kredensial langsung ke *environment variables runtime* kontainer atau ke berkas berbasis memori RAM (`tmpfs`), sehingga kredensial sama sekali tidak pernah tersimpan di *disk storage* fisik, tidak meninggalkan jejak pada citra Docker (*image layer*), dan bebas dari risiko kebocoran di Git.

\pagebreak

## **4.8 Verifikasi dan Skenario Pengujian**

\begin{itemize}
\item[\boxtimes] PostgreSQL berjalan dengan volume \texttt{pg-data}.
\item[\boxtimes] Tabel \texttt{students} dibuat otomatis dari \textit{init script}.
\item[\boxtimes] pgAdmin dapat login dan terkoneksi ke database.
\item[\boxtimes] Backup menghasilkan file \textit{dump} di direktori host.
\item[\boxtimes] Restore sudah diuji, bukan hanya diasumsikan berhasil.
\end{itemize}

\vspace{0.2cm}

| Prinsip troubleshooting: Mulai dari status container, baca logs, cek network, cek volume, lalu validasi konfigurasi. Jangan langsung menghapus volume sebelum memahami apakah data masih dibutuhkan. |
| :--- |

\vspace{0.2cm}

## **4.9 Troubleshooting dan Analisis Hasil**

| Gejala | Penyebab yang Mungkin | Tindakan Korektif |
| :--- | :--- | :--- |
| Gate berbeda antara lokal dan pipeline | Versi tool, input efektif, atau konfigurasi tidak sama | Pin versi; simpan konfigurasi efektif dan identitas artefak. |
| Service sehat tetapi security gate gagal | Healthcheck hanya memeriksa availability | Tinjau policy, scan, identity, signature, dan evidence secara terpisah. |
| Evidence tidak dapat ditelusuri | Commit, digest, waktu, atau owner tidak dicatat | Gunakan manifest evidence dan metadata yang konsisten. |
| Deployment gagal dipulihkan | Rollback, backup, atau credential rotation belum diuji | Lakukan recovery exercise dan dokumentasikan hasilnya. |

\pagebreak

\begin{center}
\textbf{\Large BAB V} \\[2pt]
\textbf{\Large KESIMPULAN}
\end{center}
\vspace{0.2cm}

Berdasarkan seluruh tahapan praktikum dan analisis yang telah dilakukan pada Bab 5, dapat disimpulkan bahwa:

1. **Pemisahan Lapisan Compute dan Storage** membuktikan bahwa kontainerisasi database relasional menuntut pemisahan mutlak antara siklus hidup kontainer (*compute*) yang bersifat fana dan media penyimpanan data (*storage*). Penggunaan *named volume* Docker terbukti menjamin data tetap persisten meskipun kontainer dimatikan, dihapus, atau diperbarui versinya.
2. **Otomatisasi Skema Initdb** menyediakan mekanisme inisialisasi skema dan data awal yang konsisten saat kluster database pertama kali dibuat, sekaligus melindungi data yang sudah ada agar tidak tertimpa pada siklus *startup* berikutnya.
3. **Pentingnya Restore Testing pada Disaster Recovery** menegaskan bahwa keberhasilan proses *backup* hanya terbukti valid apabila telah melalui tahapan uji pemulihan (*restore testing*) secara berkala. Validasi *checksum* SHA-256 dan pemulihan ke database pengujian terpisah menjamin kesiapan pemulihan sistem saat menghadapi insiden kehilangan data.
4. **Penerapan Prinsip DevSecOps** menjamin keamanan dan keandalan sistem melalui isolasi port pada antarmuka *loopback* `127.0.0.1`, isolasi jaringan internal `data-net`, pemisahan kredensial menggunakan berkas *environment variables* yang diabaikan oleh Git, serta penerapan *healthcheck* berkala.

\vspace{0.8cm}

\begin{center}
\small \textit{\textbf{Catatan \& Pernyataan Penggunaan AI:}\\Dokumen laporan ini disusun dengan bantuan AI yang digunakan secara terbatas untuk merapikan tata bahasa laporan serta men-generate catatan materi kuliah (saat Pak Ferry menjelaskan di kelas/lab) menjadi diagram visual peta konsep (mind map). Seluruh jawaban pada bagian Evaluasi dan Latihan Mandiri, analisis teknis, serta pengujian praktikum merupakan hasil pengerjaan, pemahaman mandiri, dan analisis saya secara langsung.}
\end{center}
