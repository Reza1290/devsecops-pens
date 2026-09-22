\begin{center}
\thispagestyle{empty}
\vspace*{1.2cm}

{\LARGE \textbf{LAPORAN PRAKTIKUM DEVSECOPS}}

\vspace{0.4cm}
{\Large \textbf{KONSEP CONTAINER DAN INSTALASI DOCKER}}

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

Containerization adalah teknologi untuk mengemas aplikasi dan seluruh dependensinya agar dapat berjalan secara konsisten di berbagai lingkungan. Berbeda dengan Virtual Machine yang membutuhkan sistem operasi dan kernel sendiri sehingga memakan banyak resource, container memanfaatkan fitur kernel Linux seperti namespace dan cgroup untuk mengisolasi proses dan membatasi penggunaan resource. Dalam DevSecOps, penggunaan Docker mempermudah pembuatan pipeline CI/CD yang konsisten, cepat, dan mudah direplikasi, sekaligus menuntut pemahaman terkait batasan keamanan container agar aplikasi yang dijalankan tetap aman.

## **1.2 Tujuan**

1. Menjelaskan perbedaan virtual machine dan container dari sisi isolasi, ukuran, startup time, dan overhead.  
2. Mengidentifikasi komponen arsitektur Docker seperti client, daemon, registry, image, container, network, dan volume.  
3. Menginstal Docker Engine pada Ubuntu dan mengonfigurasi hak akses user non-root.  
4. Menjalankan container dasar, memeriksa log, dan membuat custom image menggunakan Dockerfile.

\vspace{0.5cm}

\begin{center}
\textbf{\Large BAB II} \\[2pt]
\textbf{\Large DASAR TEORI}
\end{center}
\vspace{0.3cm}

## **2.1 Container dan Isolasi Kernel**

Container pada dasarnya adalah proses biasa di Linux yang diisolasi menggunakan fitur kernel, yaitu namespace dan control groups (cgroup). Namespace berfungsi membatasi apa yang bisa dilihat oleh proses di dalam container (seperti proses lain, mount point, jaringan, dan user), sedangkan cgroup bertugas membatasi dan mencatat penggunaan sumber daya fisik seperti CPU, memori, dan I/O disk agar satu container tidak mengganggu container lain atau host utama.

## **2.2 Arsitektur Docker dan Image Layer**

Docker menggunakan arsitektur client-server. Docker CLI menerima perintah dari pengguna dan mengirimkannya ke Docker daemon (dockerd) untuk dikelola. Docker image bersifat read-only dan tersusun atas beberapa layer yang terbentuk dari setiap baris instruksi pada Dockerfile. Saat container dijalankan, Docker menambahkan satu layer tipis yang bisa ditulis (writable layer) dengan mekanisme Copy-on-Write (CoW), sehingga image dasar dapat digunakan bersama oleh banyak container secara efisien.

\pagebreak

\begin{center}
\textbf{\Large BAB III} \\[2pt]
\textbf{\Large PELAKSANAAN PRAKTIKUM}
\end{center}
\vspace{0.3cm}

## **3.1 Persiapan Baseline Laboratorium**

Praktikum ini bertujuan untuk menyiapkan lingkungan kerja, melakukan instalasi Docker Engine pada host Ubuntu/Linux, serta memastikan service Docker berjalan dengan baik.

Perintah yang dijalankan pada host Linux/VM:  
```bash
# Update & instalasi repositori Docker
sudo apt update && sudo apt install -y ca-certificates curl gnupg lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalasi Docker CE & konfigurasi non-root user
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER && newgrp docker
docker version && docker run hello-world

# Uji coba container Nginx & Ubuntu
docker pull nginx:1.26
docker run -d --name web-public -p 8080:80 nginx:1.26 && docker ps
docker logs --tail 20 web-public && curl http://localhost:8080
docker pull ubuntu:22.04
docker run -it --name ubuntu-test ubuntu:22.04 /bin/bash
cat /etc/os-release
exit
docker rm -f web-public ubuntu-test

# Build custom Docker image & deployment
mkdir -p ~/docker-lab/custom-web && cd ~/docker-lab/custom-web
docker build -t pens-web:1.0 .
docker run -d --name pens-app -p 9090:80 pens-web:1.0 && curl http://localhost:9090
```

## **3.2 Dokumentasi Praktikum**

1. Update package list dan instalasi paket pendukung  
   ![][image2]  
2. Menambahkan GPG key resmi Docker  
   ![][image3]  
3. Menambahkan repository Docker ke APT sources list  
   ![][image4]  
4. Instalasi Docker Engine dan plugin pendukung  
   ![][image5]  
5. Menambahkan user ke group docker dan verifikasi dengan `groups`  
   ![][image6]  
6. docker version  
   ![][image7]  
7. docker run hello-world  
   ![][image8]  
8. docker pull nginx:1.26  
   ![][image9]  
9. docker run -d --name web-public -p 8080:80 nginx:1.26  
   ![][image10]  
10. docker ps dan docker logs  
    ![][image11]  
11. curl http://localhost:8080  
    ![][image12]  
12. docker pull ubuntu:22.04  
    ![][image13]  
13. Menjalankan container ubuntu dan cek os-release  
    ![][image14]  
14. docker rm -f web-public ubuntu-test  
    ![][image15]  
15. Menyiapkan file index.html dan Dockerfile  
    ![][image16]  
16. docker build -t pens-web:1.0 .  
    ![][image17]  
17. Menjalankan pens-app dan uji curl http://localhost:9090  
    ![][image18]

\pagebreak

## **3.3 Verifikasi dan Skenario Pengujian**

Berikut adalah matriks pengujian dan verifikasi kriteria keberhasilan praktikum Bab 2:

- [x] **Docker Engine aktif dan `docker version` menampilkan Client serta Server**  
  *Evidence*: Docker Client v29.1.3 dan Docker Server Engine v29.1.3 aktif dan berkomunikasi melalui socket `/var/run/docker.sock`.  
  ![][image7]

- [x] **User non-root dapat menjalankan `docker ps` tanpa `sudo`**  
  *Evidence*: User `ubuntu` telah dimasukkan ke dalam group `docker` (`sudo usermod -aG docker $USER`) sehingga dapat mengeksekusi `docker ps` tanpa hak akses `sudo`.  
  ![][image6]

- [x] **Container Nginx dapat diakses dari browser melalui port host**  
  *Evidence*: Container `web-public` berjalan dengan pemetaan port `-p 8080:80` dan pengujian `curl -i http://localhost:8080` mengembalikan kode status `HTTP/1.1 200 OK` (halaman *Welcome to nginx!*).  
  ![][image12]

- [x] **Image `pens-web:1.0` berhasil dibangun dan dijalankan**  
  *Evidence*: `Dockerfile` berhasil dibuild menjadi image `pens-web:1.0` dan dijalankan pada port 9090 dengan respon konten HTML `<h1>Docker Lab PENS</h1>`.  
  ![][image18]

- [x] **Mahasiswa dapat menjelaskan perbedaan `EXPOSE` dan `-p` (Publish)**  
  *Evidence*: Terverifikasi pada pembahasan teori dan analisis praktikum, di mana instruksi `EXPOSE` pada Dockerfile hanya berfungsi sebagai dokumentasi metadata deklaratif port internal container, sedangkan flag `-p` (`--publish`) pada CLI secara aktif membuat aturan forwarding/NAT iptables pada host Linux untuk memetakan port host ke port container.

\pagebreak

\begin{center}
\textbf{\Large BAB IV} \\[2pt]
\textbf{\Large HASIL DAN PEMBAHASAN}
\end{center}
\vspace{0.3cm}

## **4.1 Analisis Hasil**

Hasil pemeriksaan versi melalui `docker version` membuktikan bahwa komponen client dan server Docker sudah terpasang dan dapat diakses oleh user non-root. Eksekusi `hello-world` menunjukkan alur runtime Docker yang berhasil menarik image dari registry, membuat container, lalu berhenti saat tugasnya selesai. Pengunduhan dan eksekusi image Nginx (`web-public`) serta Ubuntu (`ubuntu-test`) membuktikan fungsi penarikan image resmi (*official library*) dari Docker Hub berjalan dengan andal. Container Nginx dan custom web (`pens-app`) tetap berjalan di background karena proses web server aktif mendengarkan port yang dipetakan (port 8080 dan 9090), dan berhasil diakses menggunakan `curl` yang mengembalikan respon HTML `200 OK`. Penggunaan base image `nginx:1.26-alpine` juga menunjukkan efisiensi layer karena ukurannya jauh lebih kecil dibanding image Linux standar.

## **4.2 Analisis Ancaman (Threat Modeling)**

| Aset | Ancaman | Jalur Serangan | Dampak Bisnis |
| ----- | ----- | ----- | ----- |
| Docker Daemon Socket (`/var/run/docker.sock`) | Host Takeover & Privilege Escalation | Akses socket Docker oleh user non-root atau container yang me-mount socket host | Pengambilalihan kontrol penuh terhadap sistem host, kebocoran data, dan manipulasi container lain |

Ancaman ini muncul karena Docker daemon secara default berjalan dengan hak akses root pada sistem host. User yang memiliki akses ke socket Docker dapat menjalankan container dengan flag `--privileged` atau me-mount root filesystem host (`/`) sehingga mendapatkan akses setara root di host. Dalam konteks DevSecOps, mitigasi dilakukan dengan membatasi akses group `docker`, tidak memasang socket ke dalam container, dan mempertimbangkan penggunaan Rootless Docker Mode.  

![][image19]

## **4.3 Analisis Masalah dan Solusi (Troubleshooting)**

Masalah yang sering muncul saat praktikum adalah *port conflict* saat menjalankan container, misalnya error `port is already allocated` pada port 8080. Cara mendiagnosisnya:

1. **Memeriksa Container Aktif**: Gunakan `docker ps` untuk melihat apakah ada container lama yang masih berjalan dan memakai port tersebut.
2. **Memeriksa Proses di Host**: Gunakan `sudo lsof -i :8080` untuk melihat aplikasi atau service lain di sistem host yang menduduki port.
3. **Solusi Penanganan**: Hentikan container lama dengan `docker rm -f <nama_container>` atau ubah konfigurasi mapping port pada host, misalnya menjadi `-p 8081:80`.

## **4.4 Analisis Keamanan dan Rekomendasi Produksi**

Berdasarkan praktikum, terdapat beberapa risiko keamanan dan rekomendasi konfigurasi untuk lingkungan produksi:

1. **Port Binding Publik**: Penggunaan `-p 8080:80` secara default mengikat port ke `0.0.0.0` (terbuka ke jaringan publik). Di lingkungan produksi, port internal sebaiknya diikat ke `127.0.0.1` dan diakses melalui reverse proxy.
2. **User Non-Root di Container**: Proses di dalam container sebaiknya tidak dijalankan sebagai root (UID 0). Pada Dockerfile perlu ditambahkan instruksi `USER` non-root untuk membatasi hak akses jika container berhasil dieksploitasi.
3. **Pinning Versi Image**: Hindari penggunaan tag `:latest` pada production. Gunakan tag versi spesifik (misalnya `nginx:1.26-alpine` atau `ubuntu:22.04`) atau digest SHA256 agar deployment bersifat *reproducible* dan terhindar dari *breaking changes*.

## **4.5 Evaluasi dan Latihan Mandiri**

**1. Mengapa penggunaan tag `latest` tidak dianjurkan untuk deployment yang harus reproducible?**  
Tag `latest` bersifat dinamis dan selalu berubah mengikuti pembaruan terbaru dari image repository. Jika base image berubah, build di waktu yang berbeda dapat menghasilkan dependensi yang berbeda atau memicu kegagalan sistem yang tidak terduga. Untuk deployment yang konsisten (*reproducible*), sebaiknya gunakan tag versi spesifik (misalnya `nginx:1.26-alpine`) atau digest SHA256.

**2. Jelaskan peran `containerd` dan `runc` dalam arsitektur Docker.**  
`containerd` bertindak sebagai pengelola *lifecycle* container tingkat tinggi (mengurus pengunduhan image, network, storage, dan pemantauan status container), sedangkan `runc` adalah komponen tingkat rendah yang bertugas langsung membuat dan menjalankan container sesuai standar OCI di atas kernel Linux.

**3. Apa konsekuensi keamanan dari memasukkan user ke group `docker`?**  
Memasukkan user ke group `docker` memberikan hak akses penuh ke socket Docker (`/var/run/docker.sock`). Hal ini setara dengan memberikan akses root karena user dapat menjalankan container dengan hak istimewa tinggi atau melakukan mount root filesystem host tanpa perlu memasukkan password `sudo`.

**4. Bandingkan layer image `nginx:1.26-alpine` dan image custom yang Anda buat.**  
Image `nginx:1.26-alpine` berisi layer dasar OS Alpine Linux dan instalasi Nginx default. Image custom `pens-web:1.0` menggunakan image tersebut sebagai base layer, kemudian menambahkan satu layer baru dari instruksi `COPY index.html` yang mengganti halaman web default dengan file HTML praktikum.

**5. Kapan sebaiknya memilih VM daripada container?**  
VM lebih tepat dipilih jika membutuhkan kernel atau sistem operasi yang berbeda dari host (misalnya menjalankan Windows di atas host Linux), memerlukan isolasi keamanan tingkat hardware yang sangat ketat (*multi-tenancy*), atau menjalankan aplikasi monolitik *legacy*. Container lebih tepat untuk aplikasi modern, *microservices*, dan pipeline CI/CD yang membutuhkan waktu mulai cepat dan konsumsi resource yang efisien.

\pagebreak

\begin{center}
\textbf{\Large BAB V} \\[2pt]
\textbf{\Large KESIMPULAN}
\end{center}
\vspace{0.3cm}

Containerization mempermudah pengelolaan dan distribusi aplikasi dengan memanfaatkan fitur kernel Linux (namespace dan cgroup) sehingga lebih ringan dan cepat dibandingkan Virtual Machine. Docker menyediakan arsitektur yang modular dan efisien berbasis image layer, di mana base image dapat digunakan bersama tanpa menduplikasi storage. Dalam praktik DevSecOps, keamanan container harus diperhatikan sejak awal, mulai dari pemilihan base image yang minimal, pembatasan hak akses user, pengelolaan port jaringan yang aman, hingga memastikan konfigurasi build dapat direplikasi dengan konsisten.

\vspace{0.3cm}

\begin{center}
\textbf{\Large BAB VI} \\[2pt]
\textbf{\Large DAFTAR PUSTAKA}
\end{center}
\vspace{0.3cm}

1. NIST SP 800-190 Application Container Security Guide.
2. Open Container Initiative (OCI) Specifications.
3. Docker Engine Documentation and Best Practices.
4. [DevOps Concept-Day2](https://github.com/ferryas-pens/devsecops/blob/main/bab-02.md)

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

[image15]: images/image15.png

[image16]: images/image16.png

[image17]: images/image17.png

[image18]: images/image18.png

[image19]: images/image19.png