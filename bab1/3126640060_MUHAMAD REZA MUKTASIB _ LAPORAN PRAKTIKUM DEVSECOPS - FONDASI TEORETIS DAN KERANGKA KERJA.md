# **LAPORAN PRAKTIKUM DEVSECOPS \- FONDASI TEORITIS DAN KERANGKA KERJA**

![][image1]

**Muhamad Reza Muktasib**  
**3126640060**  
**D4 RPL IT B**  
**POLITEKNIK ELEKTRONIKA NEGERI SURABAYA**

# **BAB I PENDAHULUAN**

# **1.1 Latar Belakang**

DevSecOps bukan sekadar penambahan alat pemindai ke dalam pipeline, melainkan sebuah sistem sosio-teknis yang melibatkan perubahan budaya, proses, dan orientasi sistem. Dalam era transformasi digital, perangkat lunak menjadi media interaksi utama bagi organisasi untuk menciptakan nilai. Gangguan layanan, kebocoran data, atau kegagalan keamanan bukan lagi sekadar masalah teknis, melainkan risiko bisnis yang signifikan yang dapat mempengaruhi kepercayaan dan kepatuhan. Dasar teori DevSecOps dimulai dari keterkaitan antara nilai bisnis, aliran perubahan perangkat lunak, dan risiko yang menyertai perubahan tersebut.

# **1.2 Tujuan**

1. Menjelaskan DevSecOps sebagai sistem sosio-teknis, bukan sekadar penambahan alat pemindai ke pipeline.  
2. Memetakan praktik DevSecOps ke kelompok praktik NIST SSDF, panduan OWASP, dan jaminan rantai pasok SLSA.  
3. Membedakan konsep *shift-left*, *shift-right*, *security gate*, serta penyediaan bukti jaminan (*evidence*) yang dapat diaudit.  
4. Menyusun baseline laboratorium dan kriteria keberhasilan eksperimen yang aman, legal, dan dapat direplikasi.

# 

# **BAB II DASAR TEORI**

# **2.1 Transformasi Digital dan DevOps**

Transformasi digital merupakan perubahan mendasar dalam cara organisasi menciptakan nilai dengan memanfaatkan data, aplikasi, dan infrastruktur sebagai satu sistem yang terintegrasi. DevOps lahir untuk memperkecil jarak antara kebutuhan bisnis yang dinamis dan kemampuan operasional TI melalui kolaborasi erat antara tim pengembangan dan operasi. Prinsip utamanya melibatkan siklus belajar yang pendek, penggunaan data umpan balik yang andal, serta pengurangan ukuran batch perubahan untuk meningkatkan kecepatan dan stabilitas delivery.

# **2.2 Evolusi SDLC**

Model pengembangan perangkat lunak telah berevolusi dari model *Waterfall* yang berurutan namun lambat dalam memberikan umpan balik, menuju model Agile yang iteratif. DevOps kemudian memperluas prinsip Agile ke seluruh aliran delivery dengan otomasi seperti CI/CD. Akhirnya, DevSecOps muncul sebagai evolusi lebih lanjut yang mengintegrasikan kontrol keamanan ke dalam lifecycle yang sama, mulai dari *threat modeling* pada tahap perencanaan hingga deteksi ancaman pada saat runtime operasional.

# 

# **BAB III PELAKSANAAN PRAKTIKUM**

# **3.1 Persiapan Baseline Laboratorium**

Praktikum ini bertujuan untuk menetapkan direktori kerja tunggal dan merekam versi komponen yang digunakan guna memastikan hasil eksperimen dapat direproduksi secara konsisten.

Perintah yang dijalankan pada host Linux/VM:  
**mkdir \-p \~/devsecops-lab/{app,policy,reports,sbom,keys}**  
**cd \~/devsecops-lab**  
**docker version**  
**docker compose version**  
**git \--version**  
**openssl version**  
**curl \--version**  
**docker info \--format '{{json .SecurityOptions}}'**

# **3.2 Dokumentasi Praktikum**

1. Login To SSH (Here i have VPS)  
   ![][image2]  
2. mkdir \-p \~/devsecops-lab/{app,policy,reports,sbom,keys}  
   ![][image3]  
3. cd \~/devsecops-lab  
   ![][image4]  
4. docker version  
   ![][image5]  
5. docker compose version  
   ![][image6]  
6. git \--version  
   ![][image7]  
7. openssl version  
   ![][image8]  
8. curl \--version  
   ![][image9]  
9. docker info \--format '{{json .SecurityOptions}}'  
   ![][image10]

# 

# **BAB IV HASIL DAN PEMBAHASAN**

# **4.1 Analisis Hasil**

Hasil pemeriksaan versi tool merupakan bukti (*evidence*) dasar untuk memastikan eksperimen dapat direproduksi di masa mendatang, karena hasil pemindaian keamanan sering kali bergantung pada versi engine dan database kerentanan yang digunakan. Pengamatan pada SecurityOptions menunjukkan mekanisme keamanan host yang tersedia, seperti *apparmor*, *seccomp*, atau *selinux*, yang menjadi dasar pengerasan (*hardening*) kontainer. Perlu dicatat bahwa perbedaan kernel atau distribusi Linux dapat memengaruhi fitur keamanan aktif yang tersedia pada daemon Docker.

# **4.2 Analisis Ancaman (Threat Modeling)**

| Aset | Ancaman | Jalur Serangan | Dampak Bisnis |
| ----- | ----- | ----- | ----- |
| Database (DB) \- Sensitive Application Data | Port Scanning & Unauthorized Access | Exposed PostgreSQL port due to misconfiguration and lack of strict authentication | Data Breach, loss of confidentiality, regulatory non-compliance, and reputational damage |

Ancaman ini muncul akibat kurangnya implementasi praktik Infrastructure-as-Code (IaC) security linting dan kurang ketatnya firewall rule pada lingkungan cloud/VM. Dalam konteks DevSecOps, mitigasi harus dilakukan dengan menerapkan principle of least privilege, enkripsi data, serta memastikan konfigurasi jaringan dikelola secara deklaratif dan otomatis.  
*![][image11]*

# **4.3 Analisis Konfigurasi Keamanan (Security Hardening)**

Berdasarkan hasil inspeksi konfigurasi kontainer (docker ps), ditemukan bahwa layanan PostgreSQL berjalan pada host 0.0.0.0:5432. Konfigurasi ini **mengekspos** database ke seluruh interface jaringan publik, yang merupakan celah keamanan signifikan (risiko brute force dan akses tanpa otorisasi). Sebagai langkah mitigasi dan penerapan prinsip "least privilege", konfigurasi port pada docker-compose.yml harus diubah menjadi 127.0.0.1:5432 agar database hanya dapat diakses secara lokal dari dalam host yang sama, bukan dari jaringan publik.

# 

# **BAB V KESIMPULAN**

DevSecOps adalah mekanisme pengelolaan risiko dan penyediaan bukti jaminan (*evidence*) yang terintegrasi pada sistem delivery perangkat lunak. Keberhasilan implementasi DevSecOps tidak diukur dari banyaknya alat pemindai yang digunakan, melainkan dari kemampuan organisasi menghasilkan perangkat lunak yang memenuhi kebutuhan bisnis sembari mengurangi risiko secara terukur dan menyediakan bukti jaminan yang dapat dipertanggungjawabkan. Penetapan baseline laboratorium pada tahap awal ini merupakan fondasi krusial untuk seluruh eksperimen keamanan berikutnya agar tetap terkendali dan dapat diaudit.

# **BAB VI DAFTAR PUSTAKA**

1. NIST SP 800-218 Secure Software Development Framework (SSDF) v1.1.  
2. OWASP DevSecOps Guideline.  
3. [DevOps Concept-Day1](https://github.com/ferryas-pens/devsecops/blob/main/bab-01.md) 

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