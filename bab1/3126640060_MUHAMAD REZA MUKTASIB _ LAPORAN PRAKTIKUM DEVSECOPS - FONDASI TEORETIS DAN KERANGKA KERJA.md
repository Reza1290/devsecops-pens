\begin{center}
\thispagestyle{empty}
\vspace*{1.2cm}

{\LARGE \textbf{LAPORAN PRAKTIKUM DEVSECOPS}}

\vspace{0.4cm}
{\Large \textbf{FONDASI TEORETIS DAN KERANGKA KERJA}}

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
\vspace{0.3cm}

## **1.1 Latar Belakang**

Transformasi digital menuntut organisasi untuk mempercepat delivery perangkat lunak guna merespons kebutuhan pasar yang dinamis. Dalam konteks ini, DevSecOps hadir sebagai evolusi dari pendekatan DevOps yang mengintegrasikan aspek keamanan sejak tahap awal perancangan (*shift-left*) hingga operasional produksi (*shift-right*). DevSecOps bukan sekadar penambahan alat pemindai (*scanner*) pada pipeline CI/CD, melainkan sebuah sistem sosio-teknis yang melibatkan transformasi budaya, kolaborasi lintas fungsi, standarisasi proses, dan penyediaan bukti jaminan (*evidence*) yang dapat diaudit. Penetapan baseline laboratorium pada tahap awal ini menjadi fondasi penting untuk memastikan seluruh eksperimen dan kontrol keamanan berikutnya dapat direproduksi secara konsisten.

## **1.2 Tujuan**

1. Menjelaskan DevSecOps sebagai sistem sosio-teknis, bukan sekadar penambahan alat pemindai ke pipeline.  
2. Memetakan praktik DevSecOps ke kelompok praktik NIST SSDF, panduan OWASP, dan jaminan rantai pasok SLSA.  
3. Membedakan konsep *shift-left*, *shift-right*, *security gate*, serta penyediaan bukti jaminan (*evidence*) yang dapat diaudit.  
4. Menyusun baseline laboratorium dan kriteria keberhasilan eksperimen yang aman, legal, dan dapat direplikasi.

\vspace{0.3cm}

\begin{center}
\textbf{\Large BAB II} \\[2pt]
\textbf{\Large DASAR TEORI}
\end{center}
\vspace{0.3cm}

## **2.1 Transformasi Digital dan DevOps**

Transformasi digital merupakan perubahan mendasar dalam cara organisasi menciptakan nilai dengan memanfaatkan data, aplikasi, dan infrastruktur sebagai satu sistem yang terintegrasi. DevOps lahir untuk memperkecil jarak antara kebutuhan bisnis yang dinamis dan kemampuan operasional TI melalui kolaborasi erat antara tim pengembangan (*development*) dan operasi (*operations*). Prinsip utamanya melibatkan siklus belajar yang pendek, penggunaan data umpan balik yang andal, serta pengurangan ukuran *batch* perubahan untuk meningkatkan kecepatan dan stabilitas *delivery*.

## **2.2 Evolusi SDLC Menuju DevSecOps**

Model pengembangan perangkat lunak telah berevolusi dari model *Waterfall* yang berurutan namun lambat dalam memberikan umpan balik, menuju model Agile yang iteratif. DevOps kemudian memperluas prinsip Agile ke seluruh aliran *delivery* dengan otomatisasi CI/CD. Akhirnya, DevSecOps muncul sebagai evolusi lebih lanjut yang mengintegrasikan kontrol keamanan ke dalam *lifecycle* yang sama. Praktik ini menerapkan prinsip *shift-left* (analisis keamanan sedini mungkin) dan *shift-right* (pemantauan serta deteksi ancaman saat *runtime* operasional) yang didukung oleh bukti jaminan (*evidence*) terverifikasi.

\pagebreak

\begin{center}
\textbf{\Large BAB III} \\[2pt]
\textbf{\Large PELAKSANAAN PRAKTIKUM}
\end{center}
\vspace{0.3cm}

## **3.1 Persiapan Baseline Laboratorium**

Praktikum ini bertujuan untuk menyiapkan struktur direktori kerja terpadu serta merekam versi seluruh komponen perangkat lunak yang digunakan guna memastikan seluruh eksperimen dapat direproduksi secara konsisten (*reproducible*).

Perintah yang dijalankan pada host Linux/VM:
```bash
# Membuat struktur direktori kerja DevSecOps
mkdir -p ~/devsecops-lab/{app,policy,reports,sbom,keys}
cd ~/devsecops-lab

# Pemeriksaan versi komponen dan runtime engine
docker version
docker compose version
git --version
openssl version
curl --version

# Pemeriksaan opsi keamanan kernel dan Docker daemon
docker info --format '{{json .SecurityOptions}}'
```

## **3.2 Dokumentasi Praktikum**

1. Akses SSH ke lingkungan virtual machine / VPS  
   ![][image2]  
2. Pembuatan struktur direktori `~/devsecops-lab/{app,policy,reports,sbom,keys}`  
   ![][image3]  
3. Berpindah ke direktori kerja utama `~/devsecops-lab`  
   ![][image4]  
4. Verifikasi instalasi dan versi `docker version`  
   ![][image5]  
5. Verifikasi versi `docker compose version`  
   ![][image6]  
6. Verifikasi versi Git dengan `git --version`  
   ![][image7]  
7. Verifikasi versi OpenSSL dengan `openssl version`  
   ![][image8]  
8. Verifikasi versi cURL dengan `curl --version`  
   ![][image9]  
9. Pemeriksaan fitur keamanan aktif host dengan `docker info --format '{{json .SecurityOptions}}'`  
   ![][image10]

\pagebreak

\begin{center}
\textbf{\Large BAB IV} \\[2pt]
\textbf{\Large HASIL DAN PEMBAHASAN}
\end{center}
\vspace{0.3cm}

## **4.1 Analisis Hasil**

Pencatatan versi perangkat lunak (Docker Engine 29.x, Docker Compose v2.x, Git, OpenSSL, dan cURL) menjadi bukti awal (*baseline evidence*) yang krusial. Hasil pemindaian kerentanan pada pipeline keamanan sangat bergantung pada versi engine, database CVE, dan parser yang aktif. Pemeriksaan `SecurityOptions` membuktikan bahwa mekanisme keamanan tingkat kernel Linux (seperti *apparmor*, *seccomp*, dan *cgroup*) telah aktif dan siap digunakan untuk membatasi hak akses serta mengisolasi container dari sistem host.

## **4.2 Analisis Ancaman (Threat Modeling)**

| Aset | Ancaman | Jalur Serangan | Dampak Bisnis |
| ----- | ----- | ----- | ----- |
| Database (DB) - Data Sensitif Aplikasi | Pemindaian Port & Akses Tidak Sah | Port PostgreSQL terbuka ke publik (`0.0.0.0:5432`) akibat salah konfigurasi | Kebocoran data (*data breach*), pelanggaran regulasi privasi, dan reputasi bisnis tercoreng |

Ancaman ini muncul akibat kelalaian dalam *port binding* pada deklarasi compose dan ketiadaan *security linting* otomatis pada IaC (*Infrastructure-as-Code*). Dalam DevSecOps, mitigasi dilakukan dengan membatasi interface ke localhost, menerapkan prinsip *least privilege*, serta menegakkan *security gate* sebelum konfigurasi dideploy.

![][image11]

## **4.3 Analisis Konfigurasi Keamanan (Security Hardening)**

Berdasarkan inspeksi terhadap konfigurasi container aktif, ditemukan potensi risiko di mana port database PostgreSQL diekspos ke `0.0.0.0:5432`. Langkah perbaikan (*hardening*) dilakukan sebagai berikut:

1. **Prinsip Least Privilege pada Jaringan**: Mengubah pemetaan port pada `docker-compose.yml` dari `0.0.0.0:5432:5432` menjadi `127.0.0.1:5432:5432` agar database hanya dapat diakses secara lokal dari dalam host.
2. **Isolasi Network Container**: Memanfaatkan user-defined bridge network internal pada Docker agar layanan aplikasi dan database berkomunikasi melalui nama DNS internal tanpa membuka port ke host publik.
3. **Pemberlakuan Security Policy**: Menerapkan rule firewall (iptables/UFW) yang ketat pada VM host untuk memblokir seluruh koneksi port database yang berasal dari luar.

## **4.4 Analisis Masalah dan Solusi (Troubleshooting)**

Dalam penyiapan baseline laboratorium, kendala operasional yang dapat terjadi meliputi:

1. **Izin Akses Daemon Docker Tanpa Sudo**: Jika user non-root tidak dapat menjalankan perintah Docker (error `permission denied`), solusinya adalah memasukkan user ke group `docker` (`sudo usermod -aG docker $USER`) dan memuat ulang group dengan `newgrp docker`.
2. **Kesesuaian Versi Komponen**: Perbedaan versi tool antara workstation lokal dan server build CI/CD dapat menyebabkan hasil pemindaian yang inkonsisten. Solusinya adalah mendokumentasikan baseline dan memanfaatkan containerized runners dengan image versi terkunci.

## **4.5 Evaluasi dan Latihan Mandiri**

**1. Mengapa DevSecOps tidak dapat direduksi menjadi penambahan scanner pada pipeline?**  
Menurut saya penambahan alat seperti *scanner* ke pipeline CI/CD hanya bentuk teknis dengan penambahan workflow dan proses, padahal tugas DevSecOps tidak hanya itu, mitigasi juga merupakan langkah awal dalam DevSecOps, mengidentifikasi *threat actor*. Terkadang penggunaan alat dapat membantu tetapi tidak bisa menemukan *edge case* dari sisi manusia itu sendiri, sehingga keamanan perlu dirancang dari awal dan menjadi tanggung jawab baik DevSecOps dan developer itu sendiri.

**2. Evidence apa yang membedakan klaim kontrol dari kontrol yang benar-benar terverifikasi?**  
Hal yang terpenting dalam *evidence* terverifikasi adalah telemetri atau metadata. Bukti hasil yang terverifikasi dari seorang ahli mungkin seperti *pentester* atau QA dapat membuat itu menjadi berbeda karena bereputasi dan berbeda dengan klaim kontrol yang tidak terverifikasi terkhususnya yang hanya asumsi.

**3. Bagaimana shared responsibility memengaruhi ownership risiko dan tindak lanjut temuan?**  
*Shared responsibility* dalam sebuah tim dibagi antara developer dan tim Ops, di mana developer memastikan tidak adanya kode yang dapat mengundang *threat* masuk, baik dalam penggunaan *3rd party library* dan lingkungan pengembangannya, sedangkan Ops berperan melindungi atau isolasi terhadap infra dan runtime dari aplikasi dengan memfasilitasi otomasi. Dengan pembagian ini, temuan kerentanan tidak lagi menjadi ajang saling lempar tanggung jawab, melainkan dikelola bersama sebagai prioritas *backlog* untuk segera ditindaklanjuti.

\vspace{0.3cm}

\begin{center}
\textbf{\Large BAB V} \\[2pt]
\textbf{\Large KESIMPULAN}
\end{center}
\vspace{0.3cm}

DevSecOps adalah mekanisme pengelolaan risiko dan penyediaan bukti jaminan (*evidence*) yang terintegrasi pada seluruh siklus hidup *delivery* perangkat lunak. Keberhasilan implementasi DevSecOps ditentukan oleh keterpaduan antara budaya kolaborasi, proses tata kelola, dan otomatisasi kontrol keamanan. Penetapan baseline laboratorium pada Bab 1 ini memberikan landasan operasional yang solid dan dapat direplikasi untuk eksperimen keamanan, pemindaian kerentanan, serta pengerasan sistem pada bab-bab berikutnya.

\vspace{0.3cm}

\begin{center}
\textbf{\Large BAB VI} \\[2pt]
\textbf{\Large DAFTAR PUSTAKA}
\end{center}
\vspace{0.3cm}

1. NIST SP 800-218 Secure Software Development Framework (SSDF) v1.1.
2. OWASP DevSecOps Guideline.
3. Supply-chain Levels for Software Artifacts (SLSA) v1.0.
4. [DevOps Concept-Day1](https://github.com/ferryas-pens/devsecops/blob/main/bab-01.md)

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