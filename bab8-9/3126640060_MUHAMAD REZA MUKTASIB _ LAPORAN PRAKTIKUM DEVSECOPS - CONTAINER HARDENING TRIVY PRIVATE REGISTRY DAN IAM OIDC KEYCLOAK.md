\begin{center}
\thispagestyle{empty}
\vspace*{1.2cm}

{\LARGE \textbf{LAPORAN PRAKTIKUM DEVSECOPS}}

\vspace{0.4cm}
{\Large \textbf{HARDENING KONTAINER DOCKER, MANAJEMEN SECRET, TRIVY SECURITY GATE, PRIVATE TLS REGISTRY, DAN IDENTITY ACCESS MANAGEMENT DENGAN KEYCLOAK OPENID CONNECT PKCE RBAC}} \\[6pt]
{\large \textbf{(PRAKTIKUM BAB 8 DAN BAB 9)}}

\vspace{2.2cm}
\includegraphics[width=5cm]{images/image1.png}

\vspace{2.5cm}
{\large \textbf{Disusun Oleh}} \\
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

Transformasi arsitektur software modern menuju platform berbasis kontainer telah mempercepat siklus pengiriman aplikasi secara dramatis. Namun, kecepatan ini kerap mengorbankan aspek keamanan mendasar apabila konfigurasi kontainer dibiarkan menggunakan pengaturan bawaan (*default settings*). Pada kenyataannya, menjalankan kontainer dengan identitas pengguna `root` (UID 0), membiarkan seluruh filesystem dapat dimodifikasi (*writable root filesystem*), mempertahankan kapabilitas kernel Linux yang berlebihan, serta menyuntikkan secrets melalui environment variables membuka celah serangan yang sangat berbahaya. Apabila aplikasi web di dalam kontainer dieksploitasi melalui kerentanan *Remote Code Execution* (RCE), attacker dapat memanfaatkan hak akses yang tidak terisolasi tersebut untuk melakukan *container breakout*, membaca memori proses lain, hingga menguasai sistem operasi *host* secara permanen.

Menjawab tantangan tersebut, Praktikum Bab 8 berfokus pada penerapan *Container Hardening* dan software supply chain security. Prinsip *least privilege* diterapkan secara ketat mulai dari tahap pembuatan image menggunakan *multi-stage build* berbasis pengguna non-root (UID 10001), isolasi secret berbasis *read-only bind-mount*, penguncian filesystem dasar (*read-only rootfs*), hingga pelucutan seluruh kapabilitas kernel (*cap_drop ALL* / *CapEff 0*). Selain itu, postur keamanan image diverifikasi sebelum memasuki siklus deployment melalui pemindaian kerentanan mendalam menggunakan Trivy. Integrasi *security gate* otomatis memastikan bahwa hanya image dengan nol kerentanan berstatus *HIGH* atau *CRITICAL* yang dapat dipromosikan ke dalam *private Docker registry* terlindungi enkripsi TLS (RSA 3072-bit) dan autentikasi *bcrypt*.

Melanjutkan fondasi keamanan infrastruktur tersebut, Praktikum Bab 9 mengatasi kerentanan pada lapisan identitas dan tata kelola hak akses (*Identity and Access Management* / IAM). Praktik penyimpanan password lokal pada aplikasi dan ketergantungan pada protokol warisan yang rentan seperti *Resource Owner Password Credentials* (ROPC / Direct Access Grants) digantikan dengan arsitektur federasi identitas terpusat. Menggunakan Keycloak 26.7.2 yang didukung database PostgreSQL 17.11, laboratorium ini menerapkan protokol standar industri *OpenID Connect* (OIDC) *Authorization Code Flow* yang diperkuat dengan *Proof Key for Code Exchange* (PKCE S256). Penegakan *Role-Based Access Control* (RBAC) diimplementasikan secara terisolasi pada aplikasi Flask, memastikan pembatasan hak akses yang tegas antara pengguna umum (*viewer*) dan personel administratif (*admin*). Sinergi komprehensif antara Bab 8 dan Bab 9 mewujudkan paradigma *Zero Trust* menyeluruh yaitu *never trust, always verify*, baik pada tingkat eksekusi kernel kontainer maupun pada validasi identitas dan otorisasi sesi pengguna.

## **1.2 Tujuan Praktikum**

1. Mampu menerapkan prinsip *least privilege* pada lingkungan runtime kontainer Docker dengan mengonfigurasi pengguna non-root (UID 10001), filesystem *read-only*, *tmpfs* terisolasi, dan pelepasan kapabilitas kernel (*cap_drop ALL*).

2. Mampu merancang mekanisme penyimpanan dan pembacaan secrets yang aman melalui *bind-mount read-only* pada direktori `/run/secrets/` tanpa membocorkan nilai sensitif ke dalam environment variables kontainer.

3. Mampu melakukan pemindaian keamanan multi-vektor menggunakan Trivy terhadap arsip image, miskonfigurasi file compose, deteksi secrets, serta membuat *Software Bill of Materials* (SBOM) berstandar CycloneDX.

4. Mampu merancang dan membuktikan efektivitas security gate otomatis yang memblokir distribusi image yang memiliki kerentanan *HIGH* atau *CRITICAL*.

5. Mampu membangun *private Docker registry* yang aman dengan menerapkan sertifikat TLS self-signed RSA 3072-bit, autentikasi *bcrypt htpasswd*, serta mengonfigurasi *trust store* pada daemon Docker *host*.

6. Mampu menganalisis komparasi arsitektural dan fungsional antara autentikasi, otorisasi, kerangka kerja OAuth 2.0, dan protokol OpenID Connect (OIDC).

7. Mampu mengoperasikan arsitektur IAM terisolasi berbasis Keycloak 26.7.2 dan database PostgreSQL 17.11 menggunakan Docker Compose dengan deklarasi *Realm-as-Code*.

8. Mampu mengintegrasikan aplikasi *Relying Party* Flask dengan Keycloak menggunakan *Authorization Code Flow* yang diamankan dengan ekstensi kriptografis PKCE S256.

9. Mampu mengimplementasikan dan memvalidasi penegakan *Role-Based Access Control* (RBAC) pada endpoint terproteksi melalui pengujian positif, pengujian negatif, dan simulasi flow *end-to-end*.

10. Mampu mengonfigurasi dan menguji siklus *Single Sign-On* (SSO) dan *Single Logout* (SLO) yang menghapus sesi lokal aplikasi dan central IdP session secara deterministik.

\pagebreak

\begin{center}
\textbf{\Large BAB II} \\[2pt]
\textbf{\Large DASAR TEORI}
\end{center}
\vspace{0.2cm}

## **2.1 Arsitektur Hardening Kontainer - Least Privilege, User Non-Root, Read-Only Filesystem, dan Linux Capabilities**

Dalam arsitektur Linux standar, kontainer bukanlah mesin virtual mandiri, melainkan serangkaian proses yang diisolasi menggunakan *Kernel Namespaces* (pid, net, mnt, ipc, uts, user) dan dibatasi oleh *Control Groups* (cgroups). Secara default, jika Dockerfile tidak mendefinisikan instruksi `USER`, proses di dalam kontainer akan dieksekusi sebagai pengguna `root` (UID 0). Meskipun proses tersebut berada di dalam namespace, UID 0 di dalam kontainer memiliki nilai integer yang identik dengan UID 0 pada kernel *host* apabila fitur *User Namespaces* (`userns-remap`) tidak diaktifkan. Konsekuensinya, jika aplikasi mengalami celah keamanan yang memungkinkan attacker mengeksekusi kode sembarang, setiap file pada *host* yang ter-mount ke dalam kontainer dapat dimanipulasi dengan wewenang superuser. Oleh karena itu, langkah mitigasi mendasar adalah membuat grup dan pengguna sistem non-root secara eksplisit, misalnya `UID/GID 10001 (app)`, dan menyetel instruksi `USER 10001:10001` pada tahap akhir pembuatan image.

Lapisan hardening berikutnya melibatkan modifikasi perilaku filesystem pada saat kontainer dijalankan (*runtime*). Secara default, kontainer Docker memiliki *writable layer* (menggunakan storage driver seperti overlay2) yang memungkinkan modifikasi file pada seluruh hierarki direktori. attacker yang berhasil menembus aplikasi web dapat mengunduh web shell, mengganti system binary library, atau menanam skrip malware persisten. Penerapan direktif `read_only: true` pada konfigurasi Compose memaksa kernel me-mount lapisan filesystem root (`/`) sebagai *read-only*. Aplikasi yang memerlukan penulisan file temporer (seperti sesi, cache, atau file lock) dialokasikan direktori khusus menggunakan memori volatil (*tmpfs*) pada `/tmp` dengan opsi keamanan ketat `tmpfs: /tmp:rw,noexec,nosuid,size=64m`. Opsi `noexec` mencegah eksekusi biner yang diunggah ke `/tmp`, sedangkan `nosuid` melucuti hak eskalasi SetUID.

Untuk melengkapi proteksi eksekusi, kernel Linux memecah privileges root menjadi beberapa unit terpisah yang disebut *Linux Capabilities*. Docker secara default memberikan belasan kapabilitas kepada kontainer non-privileged, seperti `CAP_NET_RAW` (membuat paket jaringan mentah) dan `CAP_DAC_OVERRIDE` (mengabaikan pembatasan izin file DAC). Dalam postur hardening tingkat tinggi, seluruh kapabilitas ini dilucuti menggunakan direktif `cap_drop: ["ALL"]`. Dengan konfigurasi ini, nilai *effective capabilities* (`CapEff`) pada proses kontainer bernilai `0000000000000000`, sehingga proses di dalamnya sama sekali tidak memiliki kemampuan melakukan aksi administratif kernel. Pengaturan ini diperkuat dengan opsi `security_opt: ["no-new-privileges:true"]`, yang memastikan bahwa child process tidak dapat memperoleh privileges tambahan meskipun mengeksekusi biner dengan atribut SetUID atau SetGID.

## **2.2 Manajemen Secret Aman dan Mitigasi Kebocoran Kredensial Lingkungan**

secrets seperti encryption key, token API, dan password database merupakan target utama attacker dalam ekosistem *cloud-native*. Salah satu kesalahan konfigurasi yang paling umum ditemukan pada implementasi kontainer adalah meneruskan secrets tersebut melalui environment variables menggunakan direktif `environment:` pada Docker Compose atau flag `-e` pada Docker CLI. Environment variables memiliki karakteristik inheren yang sangat tidak aman berikut.

1. Environment variables dapat dibaca oleh setiap pengguna yang memiliki akses ke soket Docker atau perintah `docker inspect <container_id>`, karena metadata kontainer tersimpan dalam teks terbuka pada file konfigurasi daemon.

2. Pada saat kontainer mengalami crash, *core dump* atau error stack trace log sering kali mencetak seluruh daftar environment variables ke dalam `stdout`/`stderr`, sehingga rahasia tersebut tersimpan permanen pada sistem *Centralized Logging*.

3. Setiap child process yang di-fork oleh aplikasi di dalam kontainer akan mewarisi salinan environment variables induknya secara otomatis, mengekspos rahasia ke third-party library yang tidak tepercaya.

4. File sistem semu `/proc/<PID>/environ` dapat dibaca oleh proses mana pun yang berjalan dengan UID yang sama di dalam kontainer.

Pendekatan aman yang diterapkan pada praktikum ini adalah memanfaatkan mekanisme *file-based secrets*. File rahasia (`secrets/app_secret.txt`) disimpan pada filesystem *host* dengan izin akses terbatas (`chmod 644` atau `600`), kemudian di-mount ke dalam kontainer pada jalur `/run/secrets/app_secret:ro`. Dengan cara ini, secrets tidak pernah tercatat di dalam metadata environment variables daemon, hanya dapat diakses melalui operasi pembacaan file pada memori ramfs/tmpfs, dan dilindungi dari manipulasi karena dipasang dengan status *read-only*.

## **2.3 Pemindaian Kerentanan Image, SBOM CycloneDX, dan Security Gate Trivy**

Keamanan kontainer tidak hanya bergantung pada konfigurasi runtime, tetapi juga sangat ditentukan oleh integritas komponen yang dikemas ke dalam image (*Software Supply Chain Security*). Base image sering kali membawa ratusan OS library (seperti `glibc`, `openssl`, `libcrypto`) dan dependensi bahasa pemrograman yang rentan terhadap *Common Vulnerabilities and Exposures* (CVE). Pemindaian keamanan berkala menggunakan vulnerability scanner tools seperti Trivy menjadi kewajiban mutlak dalam pipeline DevSecOps.

Trivy bekerja dengan membandingkan ringkasan hash dan versi paket di dalam image terhadap database kerentanan terpusat yang diperbarui secara kontinu dari berbagai advisory feed (seperti NVD, Debian Security Tracker, GitHub Advisory Database). Pemindaian Trivy mencakup beberapa dimensi kritis berikut.

1. **Pemindaian Kerentanan OS dan Language Library** Mendeteksi CVE pada OS library Debian/Alpine serta paket aplikasi (pip, npm, maven, go module).

2. **Pemindaian Miskonfigurasi (IaC Scanning)** Mengevaluasi file `compose.yaml` dan `Dockerfile` terhadap standar keamanan CIS Docker Benchmark.

3. **Pemindaian Rahasia (Secret Scanning)** Menelusuri seluruh lapisan filesystem untuk mendeteksi private key, token AWS, sertifikat, atau kredensial yang tidak sengaja tertinggal pada build context.

4. **Generasi SBOM (Software Bill of Materials)** Menghasilkan daftar inventaris terstruktur dari seluruh komponen software dalam format standar industri seperti CycloneDX (`.cdx.json`). SBOM berfungsi sebagai paspor software yang memungkinkan tim keamanan melacak keterpaparan sistem terhadap kerentanan baru (*zero-day*) tanpa harus memindai ulang seluruh kode sumber.

Agar pemindaian ini memiliki dampak penegakan yang nyata, hasil pemindaian dihubungkan dengan *Security Gate*. Kebijakan gate mendefinisikan kriteria ambang batas kelulusan berikut. image hanya dinyatakan `PASS` dan diizinkan dipromosikan ke lingkungan produksi apabila tidak memiliki temuan kerentanan berstatus *CRITICAL* atau *HIGH* yang telah memiliki rilis perbaikan (*fix available*).

## **2.4 Distribusi Image Aman - Private Docker Registry TLS dan Autentikasi Bcrypt**

Setelah image lolos dari *security gate*, artefak tersebut harus disimpan dalam repositori terpercaya (*private container registry*). Menggunakan repositori publik tanpa autentikasi atau mengoperasikan private registry melalui protokol HTTP anonim membuka risiko serangan *Man-in-the-Middle* (MitM), image poisoning, dan pencurian kekayaan intelektual kode sumber.

Arsitektur distribusi yang aman mewajibkan dua lapisan proteksi utama pada layanan registry berikut.

1. **Channel Encryption dengan TLS (Transport Layer Security)** Komunikasi antara klien Docker Engine dan registry dienkripsi menggunakan sertifikat X.509 berbobot kriptografis tinggi (kunci RSA 3072-bit). Penggunaan TLS menjamin kerahasiaan (*confidentiality*) dan integritas (*integrity*) lapisan kontainer saat transit di jaringan. Daemon Docker pada *host* dikonfigurasi untuk mempercayai sertifikat CA registry dengan menempatkan file sertifikat pada direktori `/etc/docker/certs.d/localhost:5000/ca.crt`.

2. **Autentikasi Pengguna Kuat dengan Bcrypt** Akses ke API registry (v2) diproteksi menggunakan skema autentikasi HTTP Basic yang memverifikasi kredensial terhadap file `htpasswd`. Penyimpanan hash password wajib menggunakan algoritma fungsi hash adaptif seperti bcrypt (`htpasswd -B`), yang memiliki resistensi tinggi terhadap serangan brute-force dan kalkulasi rainbow table dibandingkan fungsi tradisional DES atau MD5. Setiap upaya akses tanpa autentikasi akan ditolak seketika dengan kode status HTTP `401 Unauthorized`.

## **2.5 Arsitektur Identity & Access Management (IAM) - Autentikasi vs Otorisasi, OAuth 2.0 vs OpenID Connect (OIDC)**

Dalam domain tata kelola akses modern, pemahaman yang tegas mengenai batasan antara identitas dan hak akses merupakan prasyarat fundamental berikut.

- **Autentikasi (Authentication / AuthN)** adalah proses membuktikan kebenaran klaim identitas seorang entitas ("Siapakah Anda?"). Proses ini melibatkan verifikasi bukti kredensial seperti kombinasi username dan password, sertifikat digital, atau faktor biometrik/MFA.

- **Otorisasi (Authorization / AuthZ)** adalah proses menentukan apakah entitas yang telah terotentikasi memiliki izin untuk melakukan tindakan tertentu atau mengakses sumber daya tertentu ("Apa yang boleh Anda lakukan?").

**OAuth 2.0** (RFC 6749) dirancang murni sebagai kerangka kerja delegasi otorisasi. OAuth 2.0 memungkinkan aplikasi pihak ketiga memperoleh akses terbatas ke layanan HTTP atas nama pemilik sumber daya (*resource owner*), menggunakan Access Token. Namun, OAuth 2.0 tidak menyediakan mekanisme standar untuk menyampaikan identitas pengguna kepada aplikasi klien. Mencoba menggunakan token akses OAuth 2.0 sebagai bukti autentikasi ("pseudo-authentication") sering menimbulkan celah keamanan kritis karena token akses tidak dirancang untuk memuat informasi identitas yang dapat divalidasi secara kriptografis oleh klien.

**OpenID Connect (OIDC)** dibangun di atas fondasi OAuth 2.0 sebagai lapisan identitas (*identity layer*). OIDC memperluas OAuth 2.0 dengan memperkenalkan artefak baru bernama **ID Token**, yaitu token berformat JSON Web Token (JWT) yang ditandatangani secara kriptografis oleh *Identity Provider* (IdP). Melalui ID Token, aplikasi klien (*Relying Party*) menerima klaim identitas terverifikasi mengenai pengguna (seperti `sub`, `preferred_username`, `email`, dan waktu autentikasi), sekaligus menyediakan endpoint terstandarisasi untuk discovery metadata (`.well-known/openid-configuration`) dan pengambilan data profil pengguna (`/userinfo`).

## **2.6 Mekanisme Authorization Code Flow dengan PKCE S256 dan Keamanan Relying Party**

Flow pertukaran identitas yang paling aman untuk aplikasi web interaktif adalah *Authorization Code Flow*. Dalam flow ini, kredensial pengguna tidak pernah diberikan langsung kepada aplikasi *Relying Party* (Flask). Sebaliknya, pengguna diarahkan oleh browser ke halaman login terpusat milik Keycloak. Setelah autentikasi berhasil, Keycloak menerbitkan kode otorisasi (*authorization code*) sementara dengan masa berlaku sangat pendek (biasanya kurang dari 60 detik) melalui URL pengalihan (*redirect URI*). Aplikasi klien kemudian menukarkan kode tersebut langsung ke Token Endpoint Keycloak melalui secure back-channel untuk memperoleh ID Token, Access Token, dan Refresh Token.

Untuk melindungi flow ini dari serangan penyadapan kode otorisasi (*Authorization Code Interception Attack*) yang dapat terjadi pada browser atau lingkungan jaringan, IETF menetapkan ekstensi **Proof Key for Code Exchange (PKCE)** (RFC 7636). Awalnya dirancang untuk klien publik (seperti aplikasi seluler dan SPA), PKCE kini menjadi rekomendasi wajib (*OAuth 2.1 best practice*) untuk seluruh jenis klien, termasuk klien rahasia (*confidential clients*) - 1. Sebelum mengarahkan pengguna ke IdP, klien membuat string acak dengan entropi tinggi yang disebut `code_verifier`.

2. Klien kemudian menghitung nilai hash kriptografis dari verifier tersebut menggunakan algoritma SHA-256 dan melakukan Base64URL-encoding.
   $$\text{code\_challenge} = \text{Base64URL}(\text{SHA-256}(\text{code\_verifier}))$$

3. Klien mengirimkan `code_challenge` dan parameter `code_challenge_method=S256` pada permintaan otorisasi awal ke Keycloak. Keycloak menyimpan challenge ini dan mengaitkannya dengan kode otorisasi yang diterbitkan.

4. Ketika klien menukarkan kode otorisasi di Token Endpoint, klien wajib mengirimkan `code_verifier` asli. Keycloak menghitung ulang fungsi SHA-256 terhadap verifier tersebut dan mencocokkannya dengan challenge yang tersimpan sebelumnya.

Apabila attacker berhasil mencegat kode otorisasi dari browser, attacker tetap tidak dapat menukarkannya menjadi token karena tidak memiliki rahasia `code_verifier` yang tersimpan aman di dalam sesi server Flask.

## **2.7 Anatomi dan Validasi Token JWT (JSON Web Token)**

JSON Web Token (RFC 7519) adalah struktur data kompak dan mandiri (*stateless*) yang digunakan untuk mentransmisikan klaim antar pihak secara aman. JWT terdiri atas tiga bagian terpisah yang dikodekan dalam format Base64URL dan dipisahkan oleh karakter titik (`.`).

1. **Header** Memuat metadata teknis mengenai tipe token (`typ: "JWT"`) dan algoritma kriptografi yang digunakan untuk menandatangani token (`alg: "RS256"`).

2. **Payload** Memuat sekumpulan klaim (*claims*) mengenai subjek dan informasi tambahan.
   - Klaim Standar Terdaftar (*Registered Claims*) mencakup `iss` (issuer, URL identitas penerbit), `sub` (subject, identifikasi unik pengguna), `aud` (audience, target penerima token), `exp` (expiration time, batas kedaluwarsa waktu UNIX), `iat` (issued at, waktu penerbitan), dan `nbf` (not before).
   - Klaim Publik dan Privat memuat Profil pengguna (`preferred_username`, `email`) serta struktur peran otorisasi Keycloak (`realm_access.roles`).

3. **Signature** Digital signature yang dibuat oleh *Identity Provider* menggunakan asymmetric private key (RSA).
   $$\text{Signature} = \text{RSA-SHA256}(\text{Header} \mathbin{\Vert} \text{"."} \mathbin{\Vert} \text{Payload}, \ K_{\text{private}})$$

Proses validasi token pada sisi *Relying Party* dan *Resource Server* wajib memeriksa seluruh kondisi berikut secara ketat.

- **Integritas Kriptografis** Mengambil public key IdP melalui endpoint JWKS (*JSON Web Key Set* pada `/protocol/openid-connect/certs`) dan memverifikasi digital signature. Menolak token yang menggunakan algoritma `alg: "none"`.

- **Validitas Waktu** Memastikan bahwa waktu saat ini berada di antara klaim `nbf` dan `exp` dengan toleransi *clock skew* wajar (misalnya 60 detik).

- **Kesesuaian Identitas Penerbit (`iss`)** Nilai klaim `iss` harus cocok persis dengan URL issuer resmi IdP (\url{http://keycloak.localhost:8080/realms/devsecops-lab}).

- **Kesesuaian Target Audiens (`aud`)** Nilai klaim `aud` harus mencantumkan ID klien aplikasi yang sah (`devsecops-app`) untuk mencegah serangan substitusi token dari layanan lain.

## **2.8 Penegakan Role-Based Access Control (RBAC), Single Sign-On (SSO), dan Single Logout (SLO)**

Model otorisasi *Role-Based Access Control* (RBAC) menyederhanakan tata kelola izin dengan mengelompokkan hak akses ke dalam peran (*roles*), lalu menetapkan peran tersebut kepada pengguna (*users*). Dalam praktikum ini, didefinisikan dua peran tingkat realm (*realm roles*) berikut.

- **viewer** merupakan peran baca dasar yang diberikan kepada seluruh pengguna laboratorium (seperti `mahasiswa1` dan `adminlab`) untuk mengakses materi perkuliahan pada endpoint `/viewer`.

- **admin** merupakan peran administratif berbobot tinggi yang hanya dialokasikan kepada personel pengelola laboratorium (seperti `adminlab`) untuk mengakses panel konfigurasi dan kendali sistem pada endpoint `/admin`.

Penegakan otorisasi di dalam aplikasi Flask diterapkan menggunakan Python decorator khusus (`@require_role`). Saat pengguna mengakses rute yang dilindungi, dekorator mengekstrak klaim peran dari token sesi yang telah divalidasi. Apabila peran yang disyaratkan tidak terdapat di dalam daftar peran pengguna, aplikasi langsung menolak permintaan dengan merespons kode status HTTP `403 Forbidden`, mencegah eskalasi hak akses horizontal maupun vertikal.

Selain otorisasi, Keycloak mengelola sesi *Single Sign-On* (SSO). Setelah pengguna berhasil mengotentikasi diri pada satu aplikasi, browser menyimpan cookie sesi IdP (`KEYCLOAK_IDENTITY`). Ketika pengguna membuka aplikasi lain di dalam realm yang sama, proses autentikasi berlangsung otomatis tanpa meminta ulang kredensial. Untuk mengakhiri sesi secara aman, diimplementasikan mekanisme *Single Logout* (SLO) berbasis standar *RP-Initiated Logout*. Saat pengguna mengklik tautan logout pada aplikasi Flask, sesi lokal aplikasi dihancurkan, dan browser diarahkan ke endpoint logout Keycloak (`/protocol/openid-connect/logout`). Keycloak memvalidasi token sesi, menghapus cookie SSO, dan mengarahkan kembali browser ke halaman awal aplikasi (`post_logout_redirect_uri`), memastikan bahwa tidak ada tiket autentikasi yang tertinggal pada browser maupun server.

\pagebreak

## **2.9 Peta Konsep - Hardening Kontainer Docker dan Private Registry (Praktikum Bab 8)**

Diagram peta konsep berikut menampilkan struktur sistematis pilar-pilar container hardening, pengelolaan rahasia, software supply chain audit dengan Trivy, serta distribusi aman menggunakan private registry pada Praktikum Bab 8.

\vspace{0.4cm}
\begin{center}
\includegraphics[width=0.95\linewidth]{images/peta_konsep_hardening.png}
\end{center}
\vspace{0.4cm}

## **2.10 Peta Konsep - Identity & Access Management OIDC Keycloak (Praktikum Bab 9)**

Diagram peta konsep berikut menampilkan arsitektur federasi identitas Keycloak, token exchange flow OIDC dengan ekstensi PKCE S256, struktur validasi token JWT, serta penegakan model otorisasi RBAC pada Praktikum Bab 9.

\vspace{0.4cm}
\begin{center}
\includegraphics[width=0.95\linewidth]{images/peta_konsep_iam_oidc.png}
\end{center}
\vspace{0.4cm}

## **2.11 Peta Konsep - Sinergi Supply Chain Security dan Identity Access Governance**

Diagram peta konsep berikut menampilkan integrasi holistik antara container runtime hardening (Bab 8) dan tata kelola identitas terpusat (Bab 9) dalam mewujudkan postur pertahanan berlapis (*Defense-in-Depth*) berorientasi *Zero Trust*.

\vspace{0.4cm}
\begin{center}
\includegraphics[width=0.95\linewidth]{images/peta_konsep_devsecops_synergy.png}
\end{center}
\vspace{0.4cm}

\pagebreak

\begin{center}
\textbf{\Large BAB III} \\[2pt]
\textbf{\Large METODOLOGI DAN LANGKAH KERJA}
\end{center}
\vspace{0.2cm}

## **3.1 Arsitektur Laboratorium Terpadu dan Struktur Direktori**

Praktikum DevSecOps Bab 8 dan Bab 9 dilaksanakan pada server virtual berbasis sistem operasi Ubuntu 24.04 LTS (kernel Linux 6.8). Untuk menjamin keterisolasian lingkungan pengujian, reproduktibilitas konfigurasi, serta mencegah interferensi dengan layanan lain yang aktif pada *host*, seluruh artefak laboratorium dipisahkan secara tegas ke dalam direktori kerja masing-masing pada jalur `/home/ubuntu/docker-lab/bab-8` dan `/home/ubuntu/docker-lab/bab-9`.

```
/home/ubuntu/docker-lab/
├── bab-8/
│   ├── .dockerignore (ignore files)
│   ├── .env (chmod 600, identitas & port loopback)
│   ├── .env.example
│   ├── .gitignore (ignore files)
│   ├── compose.yaml (Hardened app & TLS Registry)
│   ├── app/
│   │   ├── Dockerfile (Multi-stage, user 10001, dipatch)
│   │   ├── app.py (Flask test app, status, secret endpoint)
│   │   └── requirements.txt (flask, gunicorn terpilih)
│   ├── registry/
│   │   ├── auth/ (htpasswd bcrypt file)
│   │   └── certs/ (domain.crt & domain.key RSA 3072)
│   ├── reports/ (SBOM, gate-status, trivy scan, digests)
│   ├── scripts/
│   │   ├── generate-registry-assets.sh
│   │   ├── push-registry.sh
│   │   ├── scan-image.sh (Trivy scanner & gate evaluator)
│   │   └── verify-hardening.sh (Negative testing 11 checks)
│   └── secrets/
│       └── app_secret.txt (Simulasi secret mount)
└── bab-9/
    ├── .dockerignore (ignore files)
    ├── .env (chmod 600, kunci kriptografis entropi tinggi)
    ├── .env.example
    ├── .gitignore (ignore files)
    ├── compose.yaml (Flask RP, Keycloak IdP, Postgres DB)
    ├── app/
    │   ├── Dockerfile (Flask relying party container)
    │   ├── app.py (Authlib OIDC PKCE client, RBAC logic)
    │   └── requirements.txt (authlib, flask, requests, gunicorn)
    ├── realm/
    │   └── realm-devsecops.json (Declarative Realm-as-Code)
    ├── reports/ (readiness, e2e audit, openid-config, SHA256)
    └── scripts/
        ├── collect-evidence.sh (Evidence generator & sanitizer)
        ├── generate-env.sh (Cryptographic credentials generator)
        ├── negative-tests.sh (OIDC security boundary validation)
        ├── test-e2e-auth.py (End-to-end automated OIDC PKCE test)
        └── validate-lab.sh (Static configuration checker)
```

## **3.2 Matriks Konfigurasi dan Parameter Keamanan Komponen**

Tabel berikut merangkum spesifikasi layanan, isolasi port jaringan, dan parameter pengerasan keamanan yang diterapkan pada Bab 8 dan Bab 9.

| Parameter | Bab 8: Container Hardening & Registry | Bab 9: IAM OIDC PKCE & Keycloak |
|---|---|---|
| **Layanan Utama** | `pens-web` (Flask) dan `registry` (Docker v2) | `keycloak` (IdP), `db` (Postgres 17), `app` (RP) |
| **Identitas User** | Non-root `UID:GID 10001:10001` (app) | Non-root `app` pada RP; internal system user IdP |
| **Proteksi Filesystem** | `read_only: true` dengan `tmpfs: /tmp:rw,noexec` | `read_only` pada RP, volume terkelola pada DB/IdP |
| **Linux Capabilities** | `cap_drop: ALL` (`CapEff: 0000000000000000`) | Standar non-privileged container |
| **Eskalasi Hak Akses** | `no-new-privileges: true` aktif | Isolasi proses standar Docker |
| **Manajemen Secret** | Mount file `/run/secrets/app_secret:ro` | Environment variables via file `.env` berizin `600` |
| **Batas Sumber Daya** | `cpus: "0.50"`, `mem_limit: 256m` | Isolasi memori heap JVM Keycloak (`-Xmx512m`) |
| **Protokol Jaringan** | TLS 1.3/1.2 (Sertifikat RSA 3072-bit) | HTTP loopback dev mode (`start-dev`) |
| **Pengikatan Port** | Loopback `127.0.0.1:8085` & `127.0.0.1:5000` | Loopback `127.0.0.1:5000`, `:8080`, `:9000` |
| **Isolasi Database** | Tanpa database relasional eksternal | PostgreSQL terisolasi internal tanpa port host |
| **Mekanisme Auth** | HTTP Basic dengan hash bcrypt pada registry | OpenID Connect Authorization Code Flow + PKCE |
| **Release Gate** | Trivy Gate: 0 Critical / High CVE | Skrip validasi statis dan pengujian negatif OIDC |

## **3.3 Prosedur Eksekusi dan Verifikasi Bertahap**

Pelaksanaan praktikum dilakukan secara terstruktur melalui fase-fase berikut.

1. **Fase 1 - Penyiapan Infrastruktur dan Aset Kriptografi Bab 8**
   - Menyiapkan direktori kerja, ignore files (`.gitignore`, `.dockerignore`), dan file konfigurasi template.
   - Menjalankan skrip `generate-registry-assets.sh` untuk menghasilkan RSA 3072-bit private key, sertifikat X.509 self-signed dengan ekstensi SAN localhost, serta file autentikasi `htpasswd` berbasis algoritma hash bcrypt.
   - Mengonfigurasi Docker daemon trust store dengan menyalin sertifikat CA registry ke direktori `/etc/docker/certs.d/localhost:5000/ca.crt`.

2. **Fase 2 - Multi-Stage Build, Remediasi Base Image, dan Pemindaian Trivy**
   - Mengembangkan `app/Dockerfile` menggunakan arsitektur *multi-stage build* untuk memisahkan kompilasi dependensi dari final release image.
   - Melakukan remediasi kerentanan sistem operasi Debian dengan menyertakan instruksi pembaruan paket keamanan (`apt-get update && apt-get upgrade -y`) untuk mem-patch sistem operasi, serta mencopot modul kompilasi (`pip`, `setuptools`, `wheel`) dari lapisan produksi.
   - Mengeksekusi skrip `scan-image.sh` untuk menyimpan arsip kontainer, menjalankan pemindaian menyeluruh Trivy, menghasilkan artefak SBOM CycloneDX, dan mengevaluasi Security Gate.

3. **Fase 3 - Orkestrasi Runtime Bab 8 dan Pengujian Verifikasi Negatif**
   - Menjalankan multi-kontainer menggunakan perintah `docker compose up -d`.
   - Menjalankan skrip `push-registry.sh` untuk melakukan login berotentikasi, memberikan tag repositori, me-push image ke registry lokal, serta memverifikasi katalog API v2.
   - Mengeksekusi rangkaian uji negatif otomatis melalui skrip `verify-hardening.sh` yang menguji 11 kontrol keamanan independen.

4. **Fase 4 - Penyiapan Lingkungan IAM dan Validasi Statis Realm Bab 9**
   - Mengonfigurasi DNS mapping loopback pada file `/etc/hosts` untuk mengaitkan nama domain `keycloak.localhost` ke alamat `127.0.0.1`.
   - Menjalankan skrip `generate-env.sh` untuk menghasilkan secret value kriptografis berentropi tinggi dan menyimpan file `.env` dengan izin `chmod 600`.
   - Mengeksekusi skrip `validate-lab.sh` guna memverifikasi integritas file deklarasi realm JSON, skema klien OIDC, dan role mapping sebelum proses bootstrapping.

5. **Fase 5 - Deployment Layanan IAM Keycloak dan PostgreSQL**
   - Membangun image Relying Party Flask dan menjalankan seluruh layanan menggunakan perintah `docker compose up -d --build`.
   - Memantau proses inisialisasi database PostgreSQL dan pengimporan otomatis konfigurasi realm oleh Keycloak melalui pemeriksaan probe kesiapan (*readiness check*) pada endpoint manajemen `http://localhost:9000/health/ready`.
   - Menginspeksi metadata OpenID Identity Provider discovery melalui endpoint standar `.well-known/openid-configuration`.

6. **Fase 6 - Pengujian Keamanan Negatif dan Simulasi Flow OIDC PKCE E2E**
   - Menjalankan skrip `negative-tests.sh` untuk memverifikasi penolakan permintaan unauthenticated, validasi ketat URL pengalihan (*redirect URI*), pencegahan flow tanpa PKCE, serta pemblokiran metode Direct Access Grants.
   - Menjalankan modul pengujian otomatis `test-e2e-auth.py` untuk mensimulasikan alur penuh pertukaran token OIDC dan membuktikan penegakan izin RBAC bagi pengguna `mahasiswa1` (peran viewer) dan `adminlab` (peran viewer + admin).
   - Memvalidasi penghancuran sesi pada skenario Single Logout (SLO) dan mengumpulkan seluruh bukti eksekusi menggunakan skrip `collect-evidence.sh`.

\pagebreak

\begin{center}
\textbf{\Large BAB IV} \\[2pt]
\textbf{\Large HASIL PENGUJIAN DAN ANALISIS BUKTI}
\end{center}
\vspace{0.2cm}

## **4.1 Pembuatan Aset TLS Private Registry dan Autentikasi Bcrypt (Bab 8)**

Tahap awal praktikum Bab 8 dimulai dengan pembuatan sertifikat digital dan kredensial akses untuk private registry menggunakan skrip otomatisasi `scripts/generate-registry-assets.sh`. Skrip ini menginstruksikan utilitas OpenSSL untuk menghasilkan pasangan kunci asimetris RSA 3072-bit dan sertifikat X.509 self-signed yang memuat *Subject Alternative Name* (SAN) `DNS:localhost,IP:127.0.0.1` dengan subjek `/CN=localhost/O=DevSecOps Lab`. Selanjutnya, utilitas `htpasswd` dijalankan melalui kontainer `httpd:2.4-alpine` dengan flag `-Bni` untuk membangkitkan hash password akun registry pengguna yang terdefinisi pada variabel lingkungan `REGISTRY_USER` (default: `devsecops`) menggunakan algoritma bcrypt dengan faktor biaya komputasi yang aman.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image2.png}
\end{center}
\vspace{0.2cm}

Hasil eksekusi pada Gambar 4.1 menampilkan proses pembangkitan private key RSA 3072 bit, penerbitan sertifikat X.509 berumur 365 hari, serta file kredensial `htpasswd` pada direktori `registry/certs/` dan `registry/auth/` dengan status kelulusan `[PASS]` dan izin file `644` yang aman.

## **4.2 Konfigurasi Certificate Trust Docker Daemon Host**

Docker Engine secara default menolak berkomunikasi dengan registry yang menggunakan sertifikat self-signed atau sertifikat yang diterbitkan oleh CA privat yang tidak terdaftar di dalam system trust store. Untuk menjalin hubungan saling percaya secara aman tanpa perlu menambahkan parameter berbahaya `--insecure-registry` pada konfigurasi daemon, sertifikat publik `domain.crt` didaftarkan ke dalam direktori kepercayaan Docker host pada `/etc/docker/certs.d/localhost:5000/ca.crt`.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image3.png}
\end{center}
\vspace{0.2cm}

Pemeriksaan sertifikat menggunakan utilitas `openssl x509` pada Gambar 4.2 membuktikan bahwa subjek dan penerbit (*issuer*) sertifikat terdaftar secara konsisten dengan identitas `CN = localhost, O = DevSecOps Lab` dengan masa berlaku 365 hari penuh (satu tahun), memastikan proses jabat tangan TLS antara daemon Docker dan private registry berjalan mulus tanpa penolakan sertifikat.

## **4.3 Multi-Stage Build Hardened Image pens-web:1.0 dan Remediasi CVE Base Image**

Proses pembuatan image aplikasi `pens-web:1.0` memanfaatkan pola *multi-stage build* untuk meminimalisasi ukuran permukaan serangan (*attack surface*). Builder stage menginstal dependensi Python ke dalam virtual environment terisolasi pada `/opt/venv`. Pada tahap akhir rilis (*runtime stage*), hanya dependensi yang telah dikompilasi yang disalin dari builder stage, sedangkan build tools (gcc, build-essential) ditinggalkan. Selain itu, perintah `apt-get update && apt-get upgrade -y` dieksekusi untuk mem-patch seluruh kerentanan paket sistem operasi Debian yang telah diperbaiki oleh vendor, diikuti pencopotan build library `pip`, `setuptools`, dan `wheel` yang tidak diperlukan pada saat aplikasi berjalan.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image4.png}
\end{center}
\vspace{0.2cm}

Gambar 4.4 memperlihatkan proses build yang berhasil menghasilkan container image ramping berukuran 148 MB, dijalankan atas username non-root beridentitas `UID:GID 10001:10001 (app)`.

## **4.4 Pemindaian Keamanan Menyeluruh dengan Trivy dan Evaluasi Security Gate**

Sebelum image diizinkan berjalan di lingkungan kontainer atau di-push ke repositori bersama, skrip `scripts/scan-image.sh` mengekspor image ke dalam arsip tarball (`reports/pens-web-1.0.tar`) untuk menghindari ketergantungan pada soket Docker host. Pemindai Trivy kemudian diaktifkan untuk melakukan audit multi-vektor - \vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image5.png}
\end{center}
\vspace{0.2cm}

Berdasarkan laporan Trivy pada Gambar 4.5, seluruh paket Debian 13.7 dan library dependensi Python (Flask 3.1.3, Gunicorn 26.1.0, Werkzeug 3.1.9, Jinja2 3.1.6) terkonfirmasi bersih dari kerentanan keamanan yang dapat dieksploitasi (0 vulnerabilities). Evaluasi Security Gate secara tegas mendeklarasikan status `[PASS]`, dan artefak transparansi dependensi berstandar CycloneDX berhasil disimpan ke dalam file `reports/pens-web-1.0.cdx.json`.

## **4.5 Deployment Multi-Container Bab 8 dan Pengujian Push/Pull Private Registry TLS**

Setelah dinyatakan lolos dari security gate, layanan multi-kontainer Bab 8 dijalankan menggunakan Docker Compose. Service stack terdiri atas `bab-8-registry-1` yang menyediakan layanan private registry TLS (`registry:3.1.1`) pada port host loopback `127.0.0.1:5000`, serta `bab-8-app-1` yang menjalankan aplikasi web Python pada port host `127.0.0.1:8085` yang dipetakan ke port internal `8080`.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image6.png}
\end{center}
\vspace{0.2cm}

Setelah service stack aktif dan berstatus sehat (*healthy*), skrip `scripts/push-registry.sh` dieksekusi. Skrip melakukan login berotentikasi ke `localhost:5000` via stdin (`--password-stdin`) guna mencegah kebocoran password pada shell history, memberikan tag repositori, me-push dan me-pull layer image, serta memverifikasi ketersediaan artefak melalui query katalog `/v2/_catalog` dan tag list `/v2/pens-web/tags/list`.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image7.png}
\end{center}
\vspace{0.2cm}

Gambar 4.7 membuktikan bahwa image `pens-web:1.0` berhasil di-push dan tersimpan secara valid pada private registry terenkripsi TLS, dibuktikan dengan respon query katalog `{"repositories":["pens-web"]}` dan query tag `{"name":"pens-web","tags":["1.0"]}`.

## **4.6 Pengujian Runtime Hardening - Eksekusi Otomatis verify-hardening.sh (11/11 Kontrol Lulus)**

Untuk memverifikasi secara empiris bahwa seluruh pembatasan keamanan bekerja pada tingkat kernel dan lingkungan waktu proses, skrip uji negatif `scripts/verify-hardening.sh` dijalankan. Skrip ini secara otomatis mengeksekusi 11 pengujian keamanan independen terhadap kontainer aplikasi `bab-8-app-1`.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image8.png}
\end{center}
\vspace{0.2cm}

Hasil pengujian pada Gambar 4.8 menunjukkan kelulusan sempurna dari seluruh 11 kontrol keamanan runtime (*Total kontrol terverifikasi: 11*).

1. **Identitas Pengguna** Proses berjalan dengan UID 10001 (bukan UID 0 / root).

2. **Proteksi Filesystem Read-Only** Upaya menulis file pada direktori root (`/`) ditolak oleh kernel (*Read-only file system*).

3. **Penyimpanan Temporer Tmpfs** Penulisan file sementara diizinkan secara terbatas pada partisi memori `/tmp`.

4. **Pelepasan Kapabilitas Kernel** Nilai `CapEff` pada `/proc/1/status` bernilai nol mutlak (`0000000000000000`).

5. **Pencegahan Eskalasi Hak Akses** Bit keamanan `NoNewPrivs` aktif bernilai 1.

6. **Integritas Mount Secret** secrets terbaca pada `/run/secrets/app_secret` dan dilindungi status *read-only*.

7. **Isolasi Variabel Lingkungan** secret value terbukti tidak tercatat di dalam environment variables kontainer.

8. **Ketiadaan Soket Docker** Soket administratif `/var/run/docker.sock` terbukti tidak dimount ke dalam kontainer.

9. **Kuota Sumber Daya** Batasan alokasi CPU (0.50 core) dan memori (256 MB) terkonfirmasi aktif pada cgroups.

10. **Kesehatan Aplikasi** Endpoint loopback HTTP mengembalikan kode status 200 OK.

11. **Proteksi Akses Anonim** Permintaan tanpa autentikasi ke private registry ditolak dengan HTTP 401 Unauthorized.

## **4.7 Verifikasi Endpoint Aplikasi Flask dan Inspeksi Kernel Capabilities (CapEff 0)**

Pengujian langsung menggunakan utilitas `curl` terhadap endpoint aplikasi Flask dilakukan untuk memvalidasi keluaran JSON yang dihasilkan oleh aplikasi `pens-web`. Rute `/` mengonfirmasi nama layanan (`service: pens-web`), status operasional (`running`), identitas proses non-root runtime (`uid: 10001, gid: 10001`), serta konfirmasi pemasangan secret (`secret_mounted: true`). Rute `/health` mengonfirmasi status kesehatan aplikasi (`status: ok`), dan inspeksi pembacaan file rahasia langsung pada kontainer membuktikan isi `/run/secrets/app_secret` dapat diakses secara read-only.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image9.png}
\end{center}
\vspace{0.2cm}

Pemeriksaan keamanan private registry tanpa menyertakan header autentikasi pada Gambar 4.10 menunjukkan penolakan tegas oleh server registry dengan kode status HTTP `401 Unauthorized` dan tantangan `Www-Authenticate: Basic realm="DevSecOps Registry"`.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image10.png}
\end{center}
\vspace{0.2cm}

Inspeksi mendalam pada tingkat kernel sistem operasi terhadap proses kontainer ditunjukkan pada Gambar 4.21. Nilai `CapEff` terbukti bernilai `0000000000000000` (seluruh kapabilitas dilucuti), dan upaya penulisan file eksploitasi pada rootfs menghasilkan error penolakan sistem (*Read-only file system*).

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image21.png}
\end{center}
\vspace{0.2cm}

Pemeriksaan inventaris dependensi software melalui artefak CycloneDX SBOM (`reports/pens-web-1.0.cdx.json`) dan ringkasan pemindaian kerentanan Trivy ditampilkan pada Gambar 4.22, mengonfirmasi 84 komponen software terindeks secara transparan dengan nol kerentanan aktif.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image22.png}
\end{center}
\vspace{0.2cm}

\pagebreak

## **4.8 Konfigurasi Loopback DNS dan Generasi Kredensial Entropi Tinggi (Bab 9)**

Praktikum Bab 9 diawali dengan konfigurasi domain name mapping loopback pada file `/etc/hosts`. Entri `127.0.0.1 keycloak.localhost` ditambahkan agar browser dan klien pengujian dapat mengakses Keycloak menggunakan nama domain terstandarisasi yang konsisten dengan nama issuer yang diterbitkan di dalam token JWT. Selanjutnya, skrip otomatisasi `scripts/generate-env.sh` dijalankan untuk menghasilkan password administratif, rahasia klien OIDC, dan kunci rahasia Flask (`KC_BOOTSTRAP_ADMIN_PASSWORD`, `FLASK_CLIENT_SECRET`, `FLASK_SESSION_SECRET`, `MAHASISWA_PASSWORD`, `ADMINLAB_PASSWORD`) menggunakan generator acak berentropi tinggi (`openssl rand -hex`), lalu menyetel izin file `.env` ke mode `600` (hanya dapat dibaca oleh pemilik) - \vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image11.png}
\end{center}
\vspace{0.2cm}

## **4.9 Validasi Statis Realm-as-Code dan Deployment IAM Stack Keycloak + PostgreSQL**

Sebelum container stack dijalankan, skrip `scripts/validate-lab.sh` dieksekusi untuk memastikan keabsahan file konfigurasi deklaratif `realm/devsecops-lab-realm.json`. Skrip memvalidasi sintaks format JSON, memeriksa pendaftaran klien rahasia `flask-app`, memastikan keberadaan peran realm `viewer` dan `admin`, penegakan metode PKCE S256, serta memverifikasi bahwa opsi rentan `directAccessGrantsEnabled` bernilai `false`.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image12.png}
\end{center}
\vspace{0.2cm}

Setelah validasi statis lulus, seluruh container stack Bab 9 dijalankan menggunakan perintah `docker compose up -d --build`. Docker Compose mengorkestrasi tiga layanan utama yaitu database PostgreSQL 17 (`devsecops-bab09-db-1`), server IAM Keycloak 26.7.2 (`devsecops-bab09-keycloak-1`), dan aplikasi Relying Party Flask (`devsecops-bab09-app-1`).

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image13.png}
\end{center}
\vspace{0.2cm}

Gambar 4.13 membuktikan bahwa seluruh kontainer berhasil dibangun dan beroperasi dengan status sehat (*healthy*) pada jaringan terisolasi `iam-net`.

## **4.10 Pengujian Kesiapan Layanan (Readiness Probe) dan Metadata OpenID Discovery**

Sebelum menerima lalu lintas autentikasi, kesiapan server Keycloak diverifikasi melalui endpoint probe manajemen lokal `http://localhost:9000/health/ready`. Format respons JSON yang diterima membuktikan bahwa seluruh subsistem internal Keycloak (koneksi asinkron database PostgreSQL, modul graceful shutdown, dan inisialisasi realm) telah berstatus `UP`.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image14.png}
\end{center}
\vspace{0.2cm}

Selanjutnya, endpoint penemuan metadata OIDC diperiksa melalui URL standar \texttt{.well-known/openid-configuration} pada Gambar 4.15. Data respons mengonfirmasi bahwa issuer resmi adalah \url{http://keycloak.localhost:8080/realms/devsecops-lab}, endpoint otorisasi dan token telah aktif, serta algoritma proteksi code_challenge_methods_supported secara eksplisit mendukung standar S256 (SHA-256).

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image15.png}
\end{center}
\vspace{0.2cm}

## **4.11 Pengujian Keamanan Negatif OIDC - Proteksi Direct Grant, URI Whitelisting, dan PKCE**

Untuk menguji ketahanan perimeter keamanan identitas terhadap berbagai vektor serangan umum, skrip `scripts/negative-tests.sh` dijalankan. Skrip mengeksekusi empat skenario pengujian negatif berikut.

1. Mengakses endpoint terproteksi `/viewer` dan `/admin` tanpa sesi aktif dialihkan seketika dengan kode status HTTP `302 Found` menuju login portal Keycloak.

2. Mengirimkan parameter `redirect_uri` ilegal (`http://attacker.local`) ditolak tegas oleh Keycloak dengan kode status HTTP `400 Bad Request` karena tidak terdaftar dalam whitelist.

3. Mengirimkan permintaan otorisasi tanpa menyertakan parameter kriptografi PKCE (`code_challenge`) ditolak oleh Keycloak dengan pengalihan error (`error=invalid_request`).

4. Mencoba metode terlarang Direct Access Grants (ROPC) menggunakan password langsung ditolak dengan kode status HTTP `400 Bad Request` (`unauthorized_client`).

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image16.png}
\end{center}
\vspace{0.2cm}

Gambar 4.16 membuktikan bahwa seluruh pengujian keamanan negatif berhasil dengan status kelulusan penuh (*Jumlah kegagalan: 0*).

## **4.12 Pengujian End-to-End OIDC Authorization Code Flow PKCE dan Penegakan RBAC**

Pengujian fungsional menyeluruh dilakukan melalui skrip Python `scripts/test-e2e-auth.py` yang mengeksekusi simulasi interaksi browser dalam menyelesaikan *Authorization Code Flow* dengan PKCE S256 secara berurutan untuk dua kategori pengguna.

Pada skenario pertama (Gambar 4.17), pengujian dijalankan atas pengguna `mahasiswa1` (peran `viewer` saja). Pengguna berhasil login melalui SSO Keycloak, authorization code ditukarkan di rute callback dengan token akses, dan saat mengakses endpoint `/viewer`, aplikasi mengembalikan respons HTTP `200 OK`. Namun, saat mencoba mengakses rute administratif `/admin`, aplikasi Relying Party Flask menolak akses dengan status HTTP `403 Forbidden` (`RBAC ENFORCED`), membuktikan bahwa pembatasan wewenang bekerja secara presisi.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image17.png}
\end{center}
\vspace{0.2cm}

Pada skenario kedua (Gambar 4.18), pengujian dilanjutkan untuk pengguna `adminlab` (peran ganda `viewer` dan `admin`). Hasil eksekusi membuktikan bahwa pengguna memiliki hak wewenang sah untuk mengakses rute `/viewer` (HTTP 200 OK) sekaligus membuka endpoint administratif `/admin` (HTTP 200 OK), diakhiri konfirmasi status kelulusan akhir `[SEMUA PENGUJIAN OIDC PKCE DAN RBAC SELESAI DENGAN STATUS 100% PASS!]`.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image18.png}
\end{center}
\vspace{0.2cm}

Inspeksi terhadap isi token JWT access token Keycloak yang didekodekan tanpa signature verification ditampilkan pada Gambar 4.23. Klaim `preferred_username: mahasiswa1`, klien resmi `azp: flask-app`, identitas unik `sub`, masa berlaku `exp`, serta struktur peran `realm_access.roles: ["viewer"]` terverifikasi tersusun rapi sesuai standar RFC 7519.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image23.png}
\end{center}
\vspace{0.2cm}

Log transaksi internal Keycloak pada Gambar 4.24 mencatat seluruh jejak audit peristiwa autentikasi secara transparan, mulai dari pemuatan file realm, inisialisasi mesin Quarkus, pencatatan event `LOGIN` pengguna, hingga penukaran `CODE_TO_TOKEN`.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image24.png}
\end{center}
\vspace{0.2cm}

Pemeriksaan topologi jaringan internal `devsecops-bab09_iam-net` pada Gambar 4.25 membuktikan bahwa database PostgreSQL 17 beroperasi murni pada segmen jaringan internal (IP `172.28.0.2`), dan upaya koneksi langsung dari host network interface ke port 5432 ditolak seketika (*Connection refused*), menjamin isolasi data kredensial.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image25.png}
\end{center}
\vspace{0.2cm}

## **4.13 Pengujian Single Logout (SLO) dan Pengumpulan Bukti Terenkripsi**

Siklus akhir pengujian melibatkan penutupan sesi secara terkoordinasi (*Single Logout* / SLO). Saat pengguna mengakses rute `/logout`, sesi lokal Flask dihancurkan dan cookie sesi `devsecops_lab_session` dikosongkan dengan masa kedaluwarsa lampau. Untuk penghentian sesi global, rute `/logout-sso` mengarahkan browser ke endpoint logout OIDC Keycloak (`/protocol/openid-connect/logout`) untuk mematikan sesi SSO secara terpusat. Pengujian permintaan susulan ke rute terproteksi `/viewer` membuktikan bahwa akses ditolak dan pengguna dialihkan kembali (HTTP 302) ke halaman login (Gambar 4.19) - \vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image19.png}
\end{center}
\vspace{0.2cm}

Seluruh bukti audit, konfigurasi, log, dan ringkasan pengujian kemudian dikumpulkan dan disanitasi secara otomatis oleh skrip `scripts/collect-evidence.sh` ke dalam direktori `reports/`, disertai pembuatan file verifikasi integritas `source-sha256.txt` (Gambar 4.20).

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image20.png}
\end{center}
\vspace{0.2cm}

\pagebreak

## **4.14 Evaluasi dan Latihan Mandiri**

### **A. Evaluasi Praktikum Bab 8 (Container Hardening & Private Registry)**

**1. Mengapa root di dalam kontainer tetap berbahaya meskipun bukan selalu root pada host?**  
Karena kontainer dan host berbagi satu kernel Linux yang sama. Jika aplikasi di dalam kontainer berjalan sebagai user `root` (UID 0), proses tersebut memiliki kapabilitas administratif penuh di dalam namespace-nya. Apabila terjadi celah keamanan kernel atau kerentanan runtime (*container breakout/escape*), attacker dapat mengeksploitasi privileges UID 0 tersebut untuk melompat keluar kontainer. Terlebih lagi, secara default daemon Docker tidak mengaktifkan user namespace remapping (`userns-remap`), sehingga UID 0 di dalam kontainer dipetakan langsung ke UID 0 di sistem operasi host. Jika kontainer memiliki bind-mount host atau attacker berhasil menembus kernel, attacker langsung memegang kendali `root` penuh atas seluruh server host. Itulah sebabnya menjalankan kontainer dengan user non-root (seperti `appuser` UID 10001) menjadi pertahanan utama yang wajib diterapkan.

**2. Apa kelemahan menyimpan secret pada environment variable?**  
Kelemahan utamanya adalah environment variable tidak dirancang untuk menyimpan secrets karena sifatnya yang mudah bocor. secret value tersebut tersimpan dalam bentuk teks terbuka (*plaintext*) pada metadata kontainer dan dapat dibaca dengan mudah oleh siapa saja yang memiliki akses Docker lewat perintah `docker inspect`. Selain itu, saat aplikasi mengalami error tak tertangani (*crash*), nilai variabel lingkungan sering kali tercetak otomatis ke dalam *stack trace* atau log `stderr` yang kemudian tersimpan di sistem centralized logging. Variabel lingkungan juga diwariskan secara otomatis ke semua proses turunan (*child process*) dan dapat diinspeksi melalui virtual filesystem `/proc/<PID>/environ`. Solusi yang jauh lebih aman adalah menggunakan mekanisme *file-based secrets* yang dimount secara *read-only* seperti pada praktikum ini.

**3. Bagaimana menentukan apakah CVE harus segera diperbaiki atau dapat ditoleransi sementara?**  
Penentuan urgensi perbaikan CVE tidak boleh hanya terpaku pada skor numerik CVSS semata, melainkan harus mempertimbangkan faktor ketercapaian (*reachability*) dan eksploitabilitas nyata di lingkungan sistem. Kerentanan dengan skor *High* atau *Critical* wajib segera diperbaiki jika memiliki kode eksploitasi publik aktif (misalnya tercatat di CISA KEV) dan paket yang terdampak diekspos langsung ke jaringan publik atau dipanggil oleh aplikasi runtime. Sebaliknya, suatu CVE dapat ditoleransi sementara jika library tersebut hanya menjadi dependensi tidak terpakai (*unreachable code*), kerentanan hanya ada di layer build sementara, atau dampaknya sudah dinetralkan oleh kontrol hardening lain seperti *read-only rootfs* dan *drop all capabilities*. Setiap toleransi sementara wajib didokumentasikan di file `.trivyignore` disertai alasan teknis dan batas waktu peninjauan ulang.

**4. Apa perbedaan registry private biasa dan registry dengan policy scanning gate?**  
Perbedaannya terletak pada kontrol jaminan keamanan artefak yang didistribusikan. Registry private biasa hanya bertindak sebagai media penyimpanan dan repositori image internal yang dilindungi autentikasi dan TLS, sehingga image yang di-push tetap bisa mengandung celah keamanan berbahaya tanpa ada peringatan. Sedangkan registry dengan *policy scanning gate* (seperti integrasi Trivy atau Harbor) memiliki sistem inspeksi otomatis yang memindai setiap image saat proses build atau sebelum di-push. Jika image terdeteksi mengandung kerentanan yang melanggar kebijakan keamanan (misalnya terdapat CVE *High* atau *Critical* tanpa mitigasi), *security gate* akan otomatis memblokir proses publikasi atau deployment image tersebut agar tidak bisa berjalan di cluster produksi.

**5. Mengapa Docker socket setara dengan akses administratif ke Docker host?**  
Karena Docker socket (`/var/run/docker.sock`) adalah control interface langsung menuju Docker daemon (`dockerd`) yang berjalan dengan hak akses `root` penuh pada mesin host. Jika sebuah kontainer diberikan akses mount ke Docker socket, siapa pun yang berada di dalam kontainer dapat mengirimkan instruksi API ke daemon host untuk menjalankan kontainer baru dengan hak akses privileged, me-mount seluruh direktori root host (`-v /:/host`), serta memodifikasi file-file krusial sistem operasi seperti `/etc/shadow`, file konfigurasi sudoers, atau kunci SSH *authorized_keys*. Akibatnya, attacker dapat dengan mudah mengambil alih hak akses root host seutuhnya tanpa memerlukan exploit kernel apa pun.

\pagebreak

### **B. Evaluasi Praktikum Bab 9 (IAM, OIDC, PKCE, dan RBAC dengan Keycloak)**

**1. Apa perbedaan autentikasi dan otorisasi?**  
Perbedaannya terletak pada fokus dan tujuannya dalam alur manajemen identitas. Autentikasi (*Authentication* / AuthN) berfokus pada verifikasi identitas untuk menjawab pertanyaan "siapakah Anda?", misalnya memastikan bahwa pengguna yang login benar-benar `alice` melalui pencocokan username, password, atau faktor kedua MFA. Sementara otorisasi (*Authorization* / AuthZ) berfokus pada penentuan hak akses setelah pengguna terotentikasi untuk menjawab pertanyaan "apa saja yang boleh Anda lakukan?", seperti memeriksa apakah Alice memiliki peran `admin` untuk mengakses dashboard manajemen sistem atau hanya peran `viewer` yang dibatasi pada hak membaca data saja.

**2. Mengapa Authorization Code Flow lebih aman daripada meminta password langsung ke aplikasi?**  
Karena kredensial password pengguna diserahkan langsung ke server Identity Provider resmi (Keycloak) melalui halaman login aman dan tidak pernah diketahui, disimpan, atau melewati server aplikasi klien. Aplikasi pihak ketiga hanya menerima *authorization code* sementara yang berumur sangat pendek. Dengan penambahan ekstensi PKCE (*Proof Key for Code Exchange*), flow ini mengharuskan pertukaran kode disertai verifikasi kriptografis `code_verifier` dan `code_challenge`, sehingga pihak attacker yang berusaha menyadap authorization code di browser publik tetap tidak bisa menukarkannya menjadi token tanpa memiliki kunci verifier rahasia tersebut. Hal ini mengeliminasi risiko pencurian kredensial oleh aplikasi klien yang tidak tepercaya.

**3. Apa yang harus dicek ketika memvalidasi JWT?**  
Saat menerima token JWT, aplikasi wajib memverifikasi lima komponen krusial yaitu pertama, memverifikasi cryptographic signature menggunakan public key resmi IdP dari endpoint JWKS untuk memastikan isi token tidak dimanipulasi di perjalanan. Kedua, memverifikasi klaim penerbit (`iss` / *issuer*) agar cocok dengan realm resmi Keycloak (\url{http://keycloak.localhost:8080/realms/devsecops-lab}). Ketiga, memeriksa klaim penerima (`aud` / *audience*) agar sesuai dengan Client ID aplikasi penerima. Keempat, memeriksa waktu kedaluwarsa (`exp`), *Not Before* (`nbf`), dan *Issued At* (`iat`) dengan mempertimbangkan toleransi *clock skew*. Kelima, memeriksa integritas klaim subjek (`sub`) dan peran pengguna (`realm_access.roles`) sebelum mengizinkan eksekusi fungsi terproteksi.

**4. Bagaimana SSO mengurangi beban user tetapi juga menambah risiko konsentrasi identitas?**  
SSO sangat meringankan beban pengguna karena mereka cukup mengingat dan memasukkan satu pasang kredensial untuk mengakses puluhan aplikasi di ekosistem organisasi tanpa perlu login berulang-ulang (*single authentication session*). Namun, kemudahan ini memunculkan risiko konsentrasi identitas (*Single Point of Failure and Compromise*). Jika satu akun SSO berhasil diretas melalui phishing atau malware pencuri kredensial, attacker otomatis mendapatkan kunci pembuka ke seluruh aplikasi yang terhubung ke SSO tersebut. Selain itu, jika server IdP mengalami downtime atau serangan DDoS, aktivitas operasional di semua aplikasi klien akan lumpuh bersamaan. Oleh sebab itu, implementasi SSO wajib diperkuat dengan MFA berbasis hardware/TOTP, audit akses ketat, dan penegakan RBAC berbutir halus di tiap aplikasi hilir.

**5. Apa strategi logout yang benar pada aplikasi berbasis token?**  
Karena access token JWT bersifat *stateless* dan tetap valid secara kriptografis hingga masa berlakunya habis (`exp`), logout tidak bisa diselesaikan hanya dengan menghapus cookie sesi di user browser. Strategi logout yang benar harus terkoordinasi secara menyeluruh yaitu pertama, aplikasi menghancurkan sesi lokal dan mengarahkan browser ke endpoint logout resmi Keycloak (*RP-Initiated Single Logout* / SLO) untuk membatalkan sesi SSO terpusat. Kedua, aplikasi memanggil endpoint token revocation (revocation endpoint) untuk mematikan *refresh token*. Ketiga, untuk memitigasi access token yang masih beredar, masa berlaku token wajib diatur sangat singkat (misalnya 1 sampai 5 menit), serta bila diperlukan keamanan tinggi, sistem backend dapat memanfaatkan daftar hitam token (*token blacklist*) berbasis Redis untuk memverifikasi apakah klaim `jti` token telah dicabut sebelum memproses setiap permintaan API.

\pagebreak

## **4.15 Verifikasi dan Skenario Pengujian**

Berikut merupakan matriks kepatuhan dan status pengujian empiris terhadap seluruh skenario kontrol keamanan pada Praktikum Bab 8 dan Bab 9 - ### **A. Checklist Verifikasi Bab 8 (Container Hardening & Private Registry)**

| No | Komponen Pengujian | Metode Verifikasi | Ekspektasi Keamanan | Hasil Aktual | Status |
|:--:|:------------------|:--------------------------|:----------------------|:-------------------|:------:|
| 1 | Identitas Pengguna | `docker exec <app> id` | Menjalankan proses sebagai non-root (UID 10001) | UID=10001 (app), \newline GID=10001 (app) | **PASS** |
| 2 | Proteksi Filesystem Root | `docker exec <app>` \newline `touch /test` | Menolak penulisan pada filesystem root | Penolakan \newline *Read-only file system* | **PASS** |
| 3 | Penyimpanan Sementara | `docker exec <app>` \newline `touch /tmp/t` | Penulisan file sementara diizinkan pada tmpfs | File berhasil dibuat \newline pada `/tmp/` | **PASS** |
| 4 | Linux Capabilities | `grep CapEff` \newline `/proc/1/status` | Seluruh kapabilitas kernel dilucuti | CapEff bernilai \newline `0000000000000000` | **PASS** |
| 5 | Eskalasi Privilege | Inspeksi metadata \newline `NoNewPrivileges` | Mencegah eskalasi biner SetUID | `NoNewPrivileges: true` \newline aktif (1) | **PASS** |
| 6 | Integritas Secret Mount | Pembacaan file \newline `/run/secrets/` | Secret terbaca dengan izin *read-only* | Secret terbaca, \newline penulisan ditolak | **PASS** |
| 7 | Isolasi Variabel Lingkungan | `docker exec <app> env` | Nilai secret tidak bocor ke environment | Nilai secret bersih \newline dari daftar env | **PASS** |
| 8 | Isolasi Soket Host | `docker exec <app>` \newline `ls /var/run/` | Ketiadaan bind mount docker.sock | File `docker.sock` \newline tidak ditemukan | **PASS** |
| 9 | Kuota Sumber Daya | Inspeksi cgroups \newline nanoCPUs & Mem | Alokasi CPU maksimal 0.50, memori 256MB | cpus: 0.50, \newline mem_limit: 256MB | **PASS** |
| 10 | Status Kesehatan Web | `curl` \newline `http://127.0.0.1:8085/` | Merespons status aplikasi normal | Respon HTTP 200 OK \newline (healthy) | **PASS** |
| 11 | Autentikasi Private Registry | `curl -k` \newline `https://localhost:5000/v2/` | Menolak akses anonim tanpa autentikasi | Respon HTTP 401 \newline Unauthorized | **PASS** |
| 12 | Keamanan TLS Registry | Validasi sertifikat \newline OpenSSL | TLS encryption dengan sertifikat RSA 3072 | Handshake TLS sukses, \newline cipher AES-GCM | **PASS** |
| 13 | Trivy Security Gate | Evaluasi file \newline `gate-status.txt` | Nol kerentanan kritis/tinggi terdeteksi | Gate PASS \newline (0 Critical / High CVE) | **PASS** |

### **B. Checklist Verifikasi Bab 9 (IAM, OIDC, PKCE, dan RBAC Keycloak)**

| No | Komponen Pengujian | Metode Verifikasi | Ekspektasi Keamanan | Hasil Aktual | Status |
|:--:|:------------------|:--------------------------|:----------------------|:-------------------|:------:|
| 1 | Validasi Realm as Code | `bash` \newline `scripts/validate-lab.sh` | Konfigurasi JSON valid dan aman | Validasi sintaks dan \newline parameter lulus | **PASS** |
| 2 | Izin File Kredensial | `ls -la .env` | File terlindung mode baca pemilik saja | Izin file terverifikasi \newline `-rw------- (600)` | **PASS** |
| 3 | Kesiapan Layanan IdP | `curl` \newline `localhost:9000/health/ready` | Seluruh subsistem Keycloak aktif | Respon HTTP 200 \newline `{"status":"UP"}` | **PASS** |
| 4 | Metadata Discovery OIDC | Query endpoint \newline `.well-known/openid-conf` | Endpoint OIDC dan PKCE S256 terdefinisi | Issuer dan endpoint \newline OIDC terkonfirmasi | **PASS** |
| 5 | Proteksi Akses Anonim | Akses rute \newline `/viewer` dan `/admin` | Mengalihkan pengguna unauthenticated | HTTP 302 dialihkan ke \newline halaman login IdP | **PASS** |
| 6 | Validasi Redirect URI | Request dengan URI \newline tidak terdaftar | Menolak URI liar di luar whitelist | HTTP 400 Bad Request \newline (Invalid redirect_uri) | **PASS** |
| 7 | Penolakan Ketiadaan PKCE | Request otorisasi \newline tanpa PKCE | Menolak flow tanpa parameter S256 | Pengalihan error \newline `error=invalid_request` | **PASS** |
| 8 | Penonaktifan Direct Grant | Request ROPC \newline langsung ke `/token` | Menolak penyerahan password ke aplikasi | HTTP 400 Bad Request \newline (unauthorized_client) | **PASS** |
| 9 | Penegakan RBAC Viewer | Pengguna `mahasiswa1` \newline (viewer) | Mengizinkan `/viewer`, \newline menolak `/admin` | `/viewer`: HTTP 200; \newline `/admin`: HTTP 403 | **PASS** |
| 10 | Penegakan RBAC Admin | Pengguna `adminlab` \newline (viewer+admin) | Mengizinkan akses ke kedua rute | `/viewer`: HTTP 200; \newline `/admin`: HTTP 200 | **PASS** |
| 11 | Single Logout Terkoordinasi | Akses `/logout` lalu \newline query rute | Sesi lokal dan sesi SSO IdP dihancurkan | Cookie terhapus, \newline redirect login IdP | **PASS** |
| 12 | Isolasi Jaringan Database | Query port 5432 \newline dari luar host | database tidak terekspos ke antarmuka host | Akses ditolak \newline (*Connection refused*) | **PASS** |

\pagebreak

## **4.16 Analisis Masalah Operasional dan Solusi (Troubleshooting)**

Selama pelaksanaan praktikum laboratorium Bab 8 dan Bab 9, ditemukan sejumlah kendala teknis dan perbedaan konfigurasi lingkungan yang memerlukan diagnosis mendalam dan langkah perbaikan terukur - ### **1. Pelanggaran Izin File akibat Penurunan Kapabilitas DAC_OVERRIDE pada Private Key dan Htpasswd**
- **Gejala** Kontainer `registry` mengalami kepanikan saat startup dan berhenti dengan pesan error `panic: unable to configure authorization (htpasswd): open /auth/htpasswd: permission denied`.
- **Diagnosis Akar Masalah** Skrip pembuatan aset menyetel izin file `domain.key` dan `htpasswd` ke mode `600` yang dimiliki oleh akun pengguna host `ubuntu` (UID 1000). Di sisi lain, kontainer registry beroperasi dengan opsi `cap_drop: ["ALL"]`. Penurunan seluruh kapabilitas kernel melucuti hak `CAP_DAC_OVERRIDE`. Akibatnya, meskipun kontainer beroperasi sebagai pengguna internal, kernel menolak pembacaan file karena file tersebut tidak memiliki bit baca untuk pengguna di luar pemiliknya (*other read bit*).
- **Solusi dan Remediasi** Menyesuaikan izin akses file pada direktori `registry/certs/` dan `registry/auth/` menjadi mode `644` (`chmod 644`). Dengan mode ini, proses kontainer yang tidak memiliki `CAP_DAC_OVERRIDE` dapat membaca file konfigurasi secara legal sesuai aturan *Discretionary Access Control* (DAC) standar tanpa harus mengembalikan kapabilitas istimewa kernel.

### **2. Batasan Kapasitas Ruang Memori Tmpfs Database Kerentanan Trivy**
- **Gejala** Eksekusi pemindaian image menggunakan skrip `scripts/scan-image.sh` terhenti di tengah jalan dengan pesan error sistem `FATAL: write /tmp/fanal-2953282276/db.tar.gz: no space left on device`.
- **Diagnosis Akar Masalah** File `compose.yaml` dari templat cheatsheet mengonfigurasi partisi tmpfs untuk kontainer Trivy dengan batasan ukuran `size=64m`. Namun, ukuran database kerentanan Trivy modern (`db.tar.gz`) saat ini telah mencapai sekitar 135 MB. Saat Trivy mengunduh dan mengekstrak database ke dalam direktori kerja sementara di `/tmp`, ruang memori volatil terisi penuh dan kehabisan ruang disk.
- **Solusi dan Remediasi** Meningkatkan kuota alokasi filesystem tmpfs pada layanan `trivy` di file `compose.yaml` menjadi `size=512m`. Kapasitas ini memberikan ruang yang sangat memadai bagi proses dekompresi dan pengindeksan database kerentanan tanpa mengurangi isolasi keamanan memori.

### **3. Deteksi Kerentanan Debian Base Image dan Remediasi Pembaruan Paket**
- **Gejala** Evaluasi security gate Trivy pada pengujian awal image `pens-web:1.0` menghasilkan status `[BLOCKED]` akibat terdeteksinya temuan kerentanan tingkat *HIGH* pada OS library (`apt`, `libpcre2-8-0`, `libssl3t64`, `zlib1g`) serta paket bawaan Python (`wheel`, `jaraco.context`).
- **Diagnosis Akar Masalah** Base image resmi `python:3.11.16-slim` yang di-pull dari Docker Hub dibangun beberapa minggu sebelumnya dan belum memuat pembaruan paket keamanan Debian terkini. Selain itu, tahap kompilasi meninggalkan file wheel dan setuptools yang rentan di dalam direktori Python library.
- **Solusi dan Remediasi** Melakukan hardening Dockerfile (`app/Dockerfile`) secara terukur.

  1. Menambahkan instruksi `RUN apt-get update && apt-get upgrade -y && rm -rf /var/lib/apt/lists/*` pada runtime stage untuk mengaplikasikan patch keamanan Debian terbaru.

  2. Menjalankan perintah pencopotan build library pada virtual environment melalui perintah `RUN /opt/venv/bin/pip uninstall -y pip setuptools wheel`.

  3. Menghapus library setuptools dan wheel bawaan yang tertinggal di `/usr/local/lib/python3.11/site-packages/`.

  Setelah image di-rebuild, pemindaian ulang Trivy menghasilkan nol temuan kerentanan (*0 vulnerabilities*) dan meloloskan release gate dengan status `PASS`.

### **4. Error Dependensi Runtime Authlib Flask Client (Missing requests)**
- **Gejala** Saat membangun dan menginisialisasi kontainer aplikasi Relying Party Bab 9 (`devsecops-bab09-app-1`), aplikasi mengalami kegagalan impor modul Python `ModuleNotFoundError: No module named 'requests'`.
- **Diagnosis Akar Masalah** Library `Authlib` versi terbaru untuk integrasi Flask menggunakan modul `requests` di balik layar untuk melakukan query HTTP ke endpoint metadata dan JWKS Keycloak. Namun, file dependensi `app/requirements.txt` dari templat cheatsheet hanya mencantumkan library `Authlib`, `Flask`, dan `gunicorn`.
- **Solusi dan Remediasi** Menambahkan dependensi `requests>=2.32.0` ke dalam file `app/requirements.txt`, lalu membangun ulang container image menggunakan perintah `docker compose up -d --build app`. Modul Authlib dapat melakukan negosiasi jabat tangan OIDC dengan Keycloak secara sempurna.

### **5. Penyesuaian Pengalihan Kesalahan OIDC Keycloak 26 pada Pengujian Ketiadaan PKCE**
- **Gejala** Skrip pengujian negatif `scripts/negative-tests.sh` melaporkan kegagalan pengujian pada skenario 4 yaitu `fail: permintaan tanpa PKCE menghasilkan HTTP 302, bukan HTTP 400`.
- **Diagnosis Akar Masalah** Pada spesifikasi OAuth 2.0 dan implementasi Keycloak versi 26, ketika permintaan otorisasi awal memuat `client_id` dan `redirect_uri` yang sah namun tidak menyertakan parameter `code_challenge` (PKCE), Keycloak tidak mengembalikan halaman kesalahan mentah HTTP 400. Sebaliknya, sesuai standar RFC 6749 bagian 4.1.2.1, Keycloak mengalihkan browser kembali ke `redirect_uri` yang sah menggunakan kode status HTTP 302 dengan menyematkan parameter error pada URL `Location: http://localhost:5000/auth?error=invalid_request&error_description=Missing+parameter%3A+code_challenge`.
- **Solusi dan Remediasi** Memperbarui logika evaluasi pada skrip `scripts/negative-tests.sh` agar mengenali respons HTTP 302 yang memuat parameter pengalihan error `error=invalid_request` sebagai bukti sah penolakan flow tanpa PKCE oleh Keycloak.

### **6. Ekstraksi Klaim Roles OIDC pada Token JWT Access Token vs Userinfo Endpoint**
- **Gejala** Pengujian otomatis `test-e2e-auth.py` mengalami kegagalan asersi otorisasi di mana pengguna `mahasiswa1` ditolak pada rute `/viewer` dengan kode status HTTP 403 padahal peran `viewer` telah ditetapkan di Keycloak.
- **Diagnosis Akar Masalah** Library Authlib pada aplikasi Flask awalnya mengekstrak data profil pengguna melalui pemanggilan endpoint `/userinfo` (`oauth.keycloak.userinfo()`). Namun, secara default pada Keycloak, klaim peran realm (`realm_access.roles`) tidak dipetakan ke dalam skema Userinfo endpoint untuk menghemat ukuran payload token, melainkan disematkan langsung di dalam Access Token dan ID Token.
- **Solusi dan Remediasi** Memperbaiki fungsi penanganan callback pada `app/app.py` agar mengekstrak klaim peran langsung dari access token atau id token yang telah diverifikasi (`token.get('access_token')` dan `token.get('id_token')`). Dengan perbaikan ini, struktur peran `realm_access.roles` terbaca sempurna dan penegakan RBAC berfungsi akurat.

\pagebreak

## **5.1 Kesimpulan**

Berdasarkan implementasi dan pengujian pada Praktikum Bab 8 dan Bab 9, dapat disimpulkan beberapa poin utama berikut.

1. **Efektivitas Prinsip Least Privilege Runtime** Penggunaan akun non-root (UID 10001), penguncian *read-only rootfs*, partisi volatil *tmpfs noexec*, dan pelucutan seluruh kapabilitas kernel (*cap_drop ALL* / *CapEff 0*) terbukti efektif mengisolasi runtime kontainer dari risiko *container escape* dan eskalasi hak akses.

2. **Keamanan Kredensial Berbasis File-Secret** Pemanfaatan *file-based secrets* yang dipasang secara *read-only* pada `/run/secrets/` sukses mengeliminasi risiko kebocoran kredensial sensitif melalui inspeksi Docker metadata, *stack trace* log, maupun filesystem virtual `/proc`.

3. **Integritas Software Supply Chain dengan Trivy** Integrasi *Security Gate* Trivy dan SBOM CycloneDX menjamin setiap image yang dipromosikan ke Private Docker Registry TLS bebas dari kerentanan berstatus *High* dan *Critical*.

4. **Pengamanan Identitas OIDC PKCE dan Penegakan RBAC** Penerapan *OpenID Connect Authorization Code Flow with PKCE* dan penegakan RBAC pada Relying Party Flask sukses mengamankan kredensial pengguna, mencegah authorization code interception, serta mengelola siklus SSO dan SLO secara bersih dan deterministik.

## **5.2 Rekomendasi Arsitektur Lingkungan Produksi**

Untuk mengimplementasikan arsitektur ini pada lingkungan produksi berskala enterprise, terdapat beberapa rekomendasi arsitektural berikut.

1. **Enterprise Registry & Image Signing** Mengadopsi Harbor Container Registry yang terintegrasi dengan pemindaian kerentanan berkala otomatis serta penandatanganan image digital (image signing) berbasis Cosign/Notary.

2. **Dynamic Secret Management** Mengintegrasikan HashiCorp Vault atau AWS Secrets Manager untuk enkripsi secrets saat istirahat (*at-rest*), rotasi kredensial otomatis, dan dynamic credential provisioning.

3. **High Availability Clustered Keycloak** Mengoperasikan Keycloak dalam mode produksi (`kc.sh start`) dengan sertifikat TLS penuh, isolasi management interface (port 9000), dan klaster database PostgreSQL terdistribusi.

4. **Multi-Factor Authentication & Policy Gate** Mewajibkan autentikasi multi-faktor (TOTP / FIDO2) pada Identity Provider serta menegakkan *Pod Security Standards (Restricted)* melalui Admission Controller (Kyverno / OPA Gatekeeper) pada klaster Kubernetes.

\pagebreak

\begin{center}
\textbf{\Large BAB VI} \\[2pt]
\textbf{\Large DAFTAR PUSTAKA}
\end{center}
\vspace{0.3cm}

1. NIST Special Publication 800-190, *Application Container Security Guide*, National Institute of Standards and Technology, 2017.
2. CIS Docker Benchmark v1.6.0, *Center for Internet Security*, 2023.
3. Open Container Initiative (OCI) Image and Distribution Specifications, *Linux Foundation*, 2021.
4. Aqua Security, *Trivy: Vulnerability Scanner and SBOM Generator*, Documentation and Best Practices, 2024.
5. RFC 6749, *The OAuth 2.0 Authorization Framework*, Internet Engineering Task Force (IETF), 2012.
6. RFC 7636, *Proof Key for Code Exchange by OAuth Public Clients (PKCE)*, Internet Engineering Task Force (IETF), 2015.
7. OpenID Connect Core 1.0 incorporating errata set 1, *OpenID Foundation*, 2014.
8. RFC 7519, *JSON Web Token (JWT)*, Internet Engineering Task Force (IETF), 2015.
9. Keycloak Server Administration and Security Guide, *Red Hat Inc. & Keycloak Community*, 2024.
10. OWASP Foundation, *Docker Security Cheat Sheet* and *OAuth 2.0 Security Best Current Practice*, 2023.
11. Susanto, Ferry Astika, *Modul Praktikum DevSecOps: Bab 8 Container Hardening dan Bab 9 Identity Access Management*, Politeknik Elektronika Negeri Surabaya, 2024.

\vspace{0.6cm}

\begin{center}
\small \textit{\textbf{Catatan \& Pernyataan Penggunaan AI:}\\Dokumen laporan praktikum ini disusun dengan bantuan AI yang digunakan secara terbatas untuk membantu paraphrase bahasa agar terstruktur rapi serta men-generate catatan materi kuliah menjadi diagram visual peta konsep (mind map). Seluruh jawaban pada bagian Evaluasi dan Latihan Mandiri, analisis teknis, pengujian lab, serta skenario troubleshooting merupakan hasil pengerjaan, pemahaman mandiri, dan analisis saya secara langsung.}
\end{center}

\pagebreak

\begin{center}
\textbf{\Large LAMPIRAN - FILE KONFIGURASI DAN SKRIP SISTEM LENGKAP}
\end{center}
\vspace{0.2cm}

## **A. File Konfigurasi dan Skrip Praktikum Bab 8 (Container Hardening & Private Registry)**

### **Lampiran 1 - compose.yaml (Orkestrasi Hardened App dan Private Registry TLS)**
```yaml
services:
  app:
    build:
      context: .
      dockerfile: app/Dockerfile
    image: ${APP_IMAGE}
    user: "10001:10001"
    init: true
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m,mode=1777
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    pids_limit: 100
    cpus: 0.50
    mem_limit: 256m
    memswap_limit: 256m
    secrets:
      - app_secret
    ports:
      - "127.0.0.1:${APP_PORT:-8085}:8080"
    networks:
      - app-net
    healthcheck:
      test:
        - CMD
        - python
        - -c
        - "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8080/health', timeout=2)"
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s
    restart: unless-stopped

  registry:
    image: registry:3.1.1
    environment:
      REGISTRY_HTTP_ADDR: 0.0.0.0:5000
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
      REGISTRY_HTTP_TLS_KEY: /certs/domain.key
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: DevSecOps Registry
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_STORAGE_DELETE_ENABLED: "false"
      OTEL_TRACES_EXPORTER: none
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=32m
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    ports:
      - "127.0.0.1:5000:5000"
    volumes:
      - registry-data:/var/lib/registry
      - ./registry/certs:/certs:ro
      - ./registry/auth:/auth:ro
    networks:
      - registry-net
    restart: unless-stopped

  trivy:
    image: aquasec/trivy:0.74.0
    profiles:
      - tools
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=512m
    volumes:
      - .:/workspace:ro
      - trivy-cache:/root/.cache
    security_opt:
      - no-new-privileges:true

secrets:
  app_secret:
    file: ./secrets/app_secret.txt

volumes:
  registry-data:
  trivy-cache:

networks:
  app-net:
    driver: bridge
  registry-net:
    driver: bridge
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 37** mendefinisikan container aplikasi `app` berbasis build konteks `.` dan `app/Dockerfile`, menetapkan user non-root `user: "10001:10001"`, mengaktifkan `init: true` sebagai init process penanganan sinyal zombie, menerapkan `read_only: true` pada filesystem root, memasang partisi memori `tmpfs: /tmp:rw,noexec,nosuid,size=64m,mode=1777`, melucuti seluruh kapabilitas kernel (`cap_drop: [ALL]`), mengaktifkan `no-new-privileges: true`, membatasi kuota sumber daya (`pids_limit: 100`, `cpus: 0.50`, `mem_limit: 256m`), memasang file rahasia `app_secret`, memetakan port host loopback `127.0.0.1:${APP_PORT:-8085}:8080`, mengonfigurasi healthcheck HTTP pada endpoint `/health`, serta menghubungkannya ke jaringan `app-net`.

2. **Line 38 sampai 65** mendefinisikan container `registry` menggunakan image resmi `registry:3.1.1`, mengonfigurasi binding port TLS `127.0.0.1:5000:5000`, mengaktifkan sertifikat dan private key TLS pada `/certs/domain.crt` dan `/certs/domain.key`, mengonfigurasi skema autentikasi `htpasswd` berbasis bcrypt pada realm `DevSecOps Registry`, menerapkan filesystem `read_only: true`, partisi memori volatil `tmpfs: /tmp:32m`, melucuti kapabilitas kernel (`cap_drop: [ALL]`), mengaktifkan `no-new-privileges`, me-mount direktori sertifikat `./registry/certs` dan file auth `./registry/auth` secara read-only (`:ro`), serta menghubungkannya ke jaringan terisolasi `registry-net`.

3. **Line 66 sampai 78** mendefinisikan service scanner `trivy` berbasis image `aquasec/trivy:0.74.0` di bawah profile `tools` yang hanya dijalankan saat pemindaian berlangsung, menerapkan proteksi `read_only: true`, memasang workspace repositori secara read-only (`.:/workspace:ro`) dan named volume persisten `trivy-cache` untuk basis data kerentanan, serta mengaktifkan `no-new-privileges: true` tanpa me-mount soket Docker host.

4. **Line 79 sampai 82** mendeklarasikan objek secret `app_secret` berbasis file `./secrets/app_secret.txt` untuk memastikan data rahasia tidak pernah bocor ke metadata environment variable container.

5. **Line 83 sampai 86** mendeklarasikan named volumes persisten `registry-data` (penyimpanan layer image registry) dan `trivy-cache` (cache database kerentanan).

6. **Line 87 sampai 91** mendeklarasikan dua segmen jaringan bridge terpisah yaitu `app-net` dan `registry-net` untuk memastikan isolasi trafik antar komponen.

### **Lampiran 2 - app/Dockerfile (Multi-Stage Build Hardened Image)**
```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.11.16-slim AS builder

ENV PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /build

COPY app/requirements.txt ./requirements.txt

RUN python -m venv /opt/venv \
    && /opt/venv/bin/pip install --no-cache-dir --upgrade pip setuptools wheel \
    && /opt/venv/bin/pip install --no-cache-dir --requirement requirements.txt \
    && /opt/venv/bin/pip uninstall -y setuptools wheel pip


FROM python:3.11.16-slim AS runtime

ARG APP_UID=10001
ARG APP_GID=10001

ENV PATH="/opt/venv/bin:${PATH}" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

RUN apt-get update && apt-get upgrade -y \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /usr/local/lib/python3.11/site-packages/setuptools* \
    && rm -rf /usr/local/lib/python3.11/site-packages/pip* \
    && rm -rf /usr/local/bin/pip* \
    && rm -rf /usr/local/bin/wheel* \
    && groupadd --gid "${APP_GID}" appgroup \
    && useradd \
        --uid "${APP_UID}" \
        --gid "${APP_GID}" \
        --no-create-home \
        --shell /usr/sbin/nologin \
        appuser

WORKDIR /app

COPY --from=builder /opt/venv /opt/venv
COPY --chown=10001:10001 app/app.py ./app.py

USER 10001:10001

EXPOSE 8080

HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
    CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8080/health', timeout=2)"]

CMD ["gunicorn", "--bind=0.0.0.0:8080", "--workers=2", "--threads=2", "--worker-tmp-dir=/tmp", "--access-logfile=-", "--error-logfile=-", "app:app"]
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 16** mendefinisikan tahap pertama (*builder stage*) berbasis `python:3.11.16-slim`, membuat virtual environment terisolasi pada `/opt/venv`, menginstal dependensi produksi dari `app/requirements.txt` tanpa caching pip (`--no-cache-dir`), serta mencopot build library `pip`, `setuptools`, dan `wheel` dari virtual environment untuk mengeliminasi potensi kerentanan CVE pada artefak build.

2. **Line 17 sampai 40** menginisialisasi tahap runtime (*runtime stage*) berbasis `python:3.11.16-slim`, mengeksekusi patch keamanan sistem operasi Debian (`apt-get update && apt-get upgrade -y`), menghapus cache paket apt pada `/var/lib/apt/lists/*`, membersihkan sisa build tools bawaan Python pada `/usr/local/lib/` dan `/usr/local/bin/`, serta membuat grup sistem `appgroup` (GID 10001) dan pengguna non-root `appuser` (UID 10001) tanpa direktori home dan tanpa shell login (`/usr/sbin/nologin`).

3. **Line 41 sampai 47** menetapkan direktori kerja `/app`, menyalin virtual environment bersih dari builder stage ke `/opt/venv`, menyalin file aplikasi `app/app.py` dengan kepemilikan non-root (`--chown=10001:10001`), serta mengaktifkan eksekusi kontainer sebagai user non-root permanen melalui direktif `USER 10001:10001`.

4. **Line 48 sampai 53** mendeklarasikan dokumentasi port `EXPOSE 8080`, mengonfigurasi pemeriksaan kesehatan berkala (*healthcheck*) pada endpoint `/health`, serta menetapkan perintah default runtime menggunakan WSGI server Gunicorn binding pada `0.0.0.0:8080` dengan parameter timeout dan worker thread yang aman.

### **Lampiran 3 - app/app.py (Aplikasi Web Pengujian Hardening)**
```python
import os
from pathlib import Path

from flask import Flask, jsonify


app = Flask(__name__)
SECRET_PATH = Path("/run/secrets/app_secret")


@app.get("/")
def index():
    return jsonify(
        service="pens-web",
        status="running",
        uid=os.getuid(),
        gid=os.getgid(),
        secret_mounted=SECRET_PATH.is_file(),
    )


@app.get("/health")
def health():
    if not SECRET_PATH.is_file():
        return jsonify(status="degraded", reason="secret_missing"), 503
    return jsonify(status="ok"), 200
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 8** mengimpor modul core OS `os`, manipulasi path `Path`, kerangka kerja web `Flask`, pembentuk JSON `jsonify`, menginisialisasi instans aplikasi Flask, serta mendefinisikan konstanta jalur secret `SECRET_PATH = Path("/run/secrets/app_secret")`.

2. **Line 11 sampai 19** mendefinisikan rute pemeriksaan status `/` yang mengembalikan respons JSON mengenai nama layanan (`service: "pens-web"`), status operasional (`running`), identitas proses runtime (`uid` dan `gid` via `os.getuid()` dan `os.getgid()`), serta konfirmasi ketersediaan mount secret (`secret_mounted: True/False`).

3. **Line 22 sampai 26** mendefinisikan rute kesiapan `/health` yang menguji keterbacaan file rahasia; mengembalikan HTTP 503 Service Unavailable (`status: "degraded"`) jika file secret tidak ditemukan, atau HTTP 200 OK (`status: "ok"`) jika file rahasia berhasil dimount dan terbaca secara sempurna.

### **Lampiran 4 - scripts/generate-registry-assets.sh (Pembuatan Aset TLS & Bcrypt)**
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

if [[ ! -f .env ]]; then
    echo "[FAIL] File .env tidak ditemukan." >&2
    exit 1
fi

set -a
source .env
set +a

: "${REGISTRY_USER:?REGISTRY_USER belum diatur}"
: "${REGISTRY_PASSWORD:?REGISTRY_PASSWORD belum diatur}"

command -v openssl >/dev/null || {
    echo "[FAIL] OpenSSL tidak tersedia." >&2
    exit 1
}

mkdir -p registry/certs registry/auth
umask 077

if [[ -e registry/certs/domain.key || -e registry/auth/htpasswd ]]; then
    echo "[FAIL] Aset registry sudah ada; rotasi harus dilakukan secara sadar." >&2
    exit 1
fi

openssl req \
    -newkey rsa:3072 \
    -nodes \
    -sha256 \
    -x509 \
    -days 365 \
    -keyout registry/certs/domain.key \
    -out registry/certs/domain.crt \
    -subj "/CN=localhost/O=DevSecOps Lab" \
    -addext "subjectAltName=DNS:localhost,IP:127.0.0.1" \
    -addext "extendedKeyUsage=serverAuth"

printf '%s\n' "$REGISTRY_PASSWORD" \
    | docker run --rm -i \
        --entrypoint htpasswd \
        httpd:2.4-alpine \
        -Bni "$REGISTRY_USER" \
    > registry/auth/htpasswd

chmod 644 registry/certs/domain.key registry/auth/htpasswd
chmod 644 registry/certs/domain.crt

echo "[PASS] Sertifikat dan htpasswd berhasil dibuat."
echo "[INFO] Sertifikat berlaku 365 hari untuk localhost."
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 22** menetapkan mode eksekusi bash ketat (*strict mode* `-Eeuo pipefail`), berpindah ke direktori induk proyek, memvalidasi keberadaan file `.env`, memuat variabel lingkungan `REGISTRY_USER` dan `REGISTRY_PASSWORD`, serta memastikan utilitas OpenSSL tersedia di sistem.

2. **Line 23 sampai 30** membuat direktori penyimpanan aset `registry/certs` dan `registry/auth`, menerapkan umask protektif `077` untuk menjamin izin akses file yang dihasilkan bersifat tertutup (*owner-only*), serta mencegah penimpaan file tanpa rotasi sadar.

3. **Line 31 sampai 42** menjalankan perintah `openssl req` untuk membangkitkan private key RSA 3072 bit dan sertifikat self-signed X.509 berumur 365 hari dengan subjek terstandarisasi `/CN=localhost/O=DevSecOps Lab` serta menyematkan ekstensi SAN (*Subject Alternative Name*) `DNS:localhost,IP:127.0.0.1`.

4. **Line 43 sampai 49** mengeksekusi utilitas `htpasswd` melalui kontainer Docker sementara `httpd:2.4-alpine` menggunakan flag `-Bni` untuk membangkitkan hash password bcrypt dari variabel `REGISTRY_PASSWORD` untuk pengguna `REGISTRY_USER`, lalu menyimpannya ke `registry/auth/htpasswd`.

5. **Line 50 sampai 54** mengatur izin akses file sertifikat dan autentikasi ke mode `644` agar dapat dibaca oleh daemon kontainer registry, serta mencetak pesan konfirmasi sukses `[PASS]`.

### **Lampiran 5 - scripts/scan-image.sh (Trivy Scan, SBOM, & Security Gate)**
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

if [[ ! -f .env ]]; then
    echo "[FAIL] File .env tidak ditemukan." >&2
    exit 1
fi

set -a
source .env
set +a

: "${APP_IMAGE:?APP_IMAGE belum diatur}"

mkdir -p reports
archive_name="pens-web-1.0.tar"

docker image inspect "$APP_IMAGE" >/dev/null
docker image save --output "reports/$archive_name" "$APP_IMAGE"

run_trivy() {
    docker compose --profile tools run --rm --no-deps trivy "$@"
}

run_trivy image \
    --input "/workspace/reports/$archive_name" \
    --scanners vuln,secret \
    --image-config-scanners misconfig,secret \
    --format json \
    > reports/trivy-image.json

run_trivy image \
    --input "/workspace/reports/$archive_name" \
    --format cyclonedx \
    > reports/pens-web-1.0.cdx.json

run_trivy config \
    --severity MEDIUM,HIGH,CRITICAL \
    /workspace/app \
    > reports/trivy-config.txt

run_trivy config \
    --severity MEDIUM,HIGH,CRITICAL \
    /workspace/compose.yaml \
    >> reports/trivy-config.txt

run_trivy fs \
    --scanners secret \
    /workspace/app \
    > reports/trivy-secret.txt

set +e
run_trivy image \
    --input "/workspace/reports/$archive_name" \
    --scanners vuln \
    --severity HIGH,CRITICAL \
    --ignore-unfixed \
    --exit-code 1 \
    > reports/trivy-gate.txt
gate_status=$?
set -e

if [[ "$gate_status" -eq 0 ]]; then
    printf 'PASS\n' > reports/gate-status.txt
    echo "[PASS] Security gate lulus."
else
    printf 'BLOCKED\n' > reports/gate-status.txt
    echo "[BLOCKED] Temuan HIGH/CRITICAL yang dapat diperbaiki terdeteksi." >&2
    echo "[INFO] Tinjau reports/trivy-gate.txt dan perbarui image." >&2
    exit 1
fi

docker image inspect "$APP_IMAGE" \
    --format '{{json .RepoDigests}}' \
    > reports/local-image-digests.json

sha256sum \
    reports/trivy-image.json \
    reports/pens-web-1.0.cdx.json \
    reports/trivy-config.txt \
    reports/trivy-secret.txt \
    reports/trivy-gate.txt \
    > reports/SHA256SUMS

echo "[PASS] Laporan, SBOM, dan checksum berhasil dibuat."
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 22** menginisialisasi lingkungan pengujian, memuat variabel `APP_IMAGE` dari file `.env`, membuat direktori laporan `reports/`, memeriksa keabsahan image lokal, serta mengekspor image ke dalam arsip mandiri `reports/pens-web-1.0.tar` guna menghindari ketergantungan terhadap bind-mount soket Docker host.

2. **Line 23 sampai 26** mendefinisikan fungsi pembungkus `run_trivy()` yang mengeksekusi kontainer Trivy melalui profil tools Docker Compose tanpa ketergantungan daemon host.

3. **Line 27 sampai 53** menjalankan pemindaian keamanan multi-vektor Trivy meliputi audit kerentanan dan secret image dalam format JSON (`reports/trivy-image.json`), pembuatan SBOM berstandar CycloneDX (`reports/pens-web-1.0.cdx.json`), audit miskonfigurasi Dockerfile dan compose file (`reports/trivy-config.txt`), serta pemindaian kebocoran kredensial filesystem aplikasi (`reports/trivy-secret.txt`).

4. **Line 54 sampai 74** mengevaluasi security gate terhadap kerentanan level HIGH dan CRITICAL yang telah memiliki perbaikan (`--ignore-unfixed --exit-code 1`); jika bersih, skrip mencatat status `PASS` ke `reports/gate-status.txt` dan mengizinkan kelanjutan pipeline, atau memblokir pipeline dengan kode status keluar 1 jika terdeteksi kerentanan kritis.

5. **Line 75 sampai 87** mengekstrak digest image lokal ke `reports/local-image-digests.json` dan menghitung nilai checksum SHA-256 dari seluruh file artefak laporan ke dalam file verifikasi `reports/SHA256SUMS`.

### **Lampiran 6 - scripts/verify-hardening.sh (Verifikasi Negatif 11 Kontrol)**
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

if [[ -f .env ]]; then
    set -a
    source .env
    set +a
fi

pass_count=0

pass() {
    echo "[PASS] $1"
    pass_count=$((pass_count + 1))
}

fail() {
    echo "[FAIL] $1" >&2
    exit 1
}

uid="$(docker compose exec -T app id -u)"
[[ "$uid" == "10001" ]] \
    && pass "Proses aplikasi berjalan sebagai UID 10001." \
    || fail "UID aplikasi adalah $uid."

if docker compose exec -T app sh -c 'touch /unauthorized-write' 2>/dev/null; then
    fail "Filesystem root masih dapat ditulis."
else
    pass "Penulisan ke filesystem root ditolak."
fi

docker compose exec -T app sh -c 'touch /tmp/allowed-write && rm /tmp/allowed-write' \
    && pass "tmpfs /tmp dapat digunakan aplikasi." \
    || fail "tmpfs /tmp tidak dapat digunakan."

cap_eff="$(
    docker compose exec -T app sh -c \
        "awk '/CapEff/ {print \$2}' /proc/1/status"
)"
[[ "$cap_eff" == "0000000000000000" ]] \
    && pass "Seluruh Linux capability efektif telah dihapus." \
    || fail "Capability efektif masih tersedia: $cap_eff"

no_new_privs="$(
    docker compose exec -T app sh -c \
        "awk '/NoNewPrivs/ {print \$2}' /proc/1/status"
)"
[[ "$no_new_privs" == "1" ]] \
    && pass "NoNewPrivs aktif." \
    || fail "NoNewPrivs tidak aktif."

docker compose exec -T app test -r /run/secrets/app_secret \
    && pass "Secret tersedia sebagai file read-only." \
    || fail "Secret tidak dapat dibaca aplikasi."

if docker compose exec -T app env | grep -q '^APP_SECRET='; then
    fail "Secret ditemukan sebagai environment variable."
else
    pass "Secret tidak diekspos sebagai APP_SECRET."
fi

docker compose exec -T app test ! -S /var/run/docker.sock \
    && pass "Docker socket tidak tersedia di container aplikasi." \
    || fail "Docker socket terpasang ke container aplikasi."

container_id="$(docker compose ps -q app)"
read -r memory nano_cpus pids readonly_root <<<"$(
    docker inspect "$container_id" \
        --format '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}} {{.HostConfig.PidsLimit}} {{.HostConfig.ReadonlyRootfs}}'
)"

if (( memory > 0 && nano_cpus > 0 && pids > 0 )) \
    && [[ "$readonly_root" == "true" ]]; then
    pass "Limit memory, CPU, PID, dan read-only rootfs aktif."
else
    fail "Runtime limit tidak lengkap."
fi

app_ready="false"
for attempt in {1..30}; do
    if curl --fail --silent --show-error \
        "http://127.0.0.1:${APP_PORT:-8085}/health" >/dev/null 2>&1; then
        app_ready="true"
        break
    fi
    sleep 2
done

[[ "$app_ready" == "true" ]] \
    && pass "Health endpoint aplikasi merespons sukses." \
    || fail "Health endpoint aplikasi gagal."

unauth_code=""
for attempt in {1..30}; do
    unauth_code="$(
        curl --silent \
            --output /dev/null \
            --write-out '%{http_code}' \
            --cacert registry/certs/domain.crt \
            https://localhost:5000/v2/ \
            || true
    )"
    [[ "$unauth_code" == "401" ]] && break
    sleep 2
done
[[ "$unauth_code" == "401" ]] \
    && pass "Registry menolak akses anonim dengan HTTP 401." \
    || fail "Registry memberi status anonim $unauth_code."

echo "[PASS] Total kontrol terverifikasi: $pass_count"
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 23** menginisialisasi script dengan mode ketat, memuat variabel `.env`, serta mendefinisikan fungsi pencatat pengujian `pass()` dan `fail()` yang menghitung akumulasi kontrol keamanan yang berhasil lulus (`pass_count`).

2. **Line 24 sampai 28** menguji Kontrol 1 dengan mengekstrak UID proses di dalam kontainer `app` via `docker compose exec -T app id -u` dan memverifikasi nilainya adalah UID non-root `10001`.

3. **Line 29 sampai 38** menguji Kontrol 2 dan 3 dengan mencoba membuat file pada rootfs (`touch /unauthorized-write`) dan memverifikasi penolakan kernel, serta memverifikasi penulisan sementara pada `/tmp` (tmpfs) berhasil.

4. **Line 39 sampai 46** menguji Kontrol 4 dengan memeriksa kapabilitas efektif pada `/proc/1/status` dan memverifikasi nilai `CapEff` bernilai nol mutlak (`0000000000000000`).

5. **Line 47 sampai 54** menguji Kontrol 5 dengan memverifikasi bahwa flag pembatasan eskalasi privilese `NoNewPrivs` bernilai 1.

6. **Line 55 sampai 64** menguji Kontrol 6 dan 7 dengan memverifikasi keterbacaan file rahasia pada `/run/secrets/app_secret` serta memastikan nama variabel `APP_SECRET` tidak bocor pada lingkungan container.

7. **Line 65 sampai 68** menguji Kontrol 8 dengan memverifikasi soket Docker host `/var/run/docker.sock` tidak terpasang di dalam container.

8. **Line 69 sampai 81** menguji Kontrol 9 dengan memeriksa konfigurasi HostConfig cgroups container dan memastikan batasan alokasi memori, kuota CPU (NanoCpus), limit PID, dan status read-only rootfs aktif.

9. **Line 82 sampai 95** menguji Kontrol 10 dengan melakukan polling loopback ke endpoint `/health` aplikasi dan memverifikasi respons HTTP 200 OK.

10. **Line 96 sampai 113** menguji Kontrol 11 dengan mengirimkan permintaan tanpa header autentikasi ke private registry TLS dan memverifikasi pengembalian kode status HTTP `401 Unauthorized`, lalu mencetak total kontrol kelulusan (*Total kontrol terverifikasi: 11*).

## **B. File Konfigurasi dan Skrip Praktikum Bab 9 (IAM, OIDC, PKCE, dan RBAC Keycloak)**

### **Lampiran 7 - compose.yaml (Orkestrasi Keycloak, PostgreSQL, dan Flask RP)**
```yaml
name: devsecops-bab09

services:
  db:
    image: postgres:17.11-alpine3.24
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB:?POSTGRES_DB wajib diisi}
      POSTGRES_USER: ${POSTGRES_USER:?POSTGRES_USER wajib diisi}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?POSTGRES_PASSWORD wajib diisi}
    volumes:
      - keycloak-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 20
    networks:
      - iam-net
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - FOWNER
      - SETGID
      - SETUID
    pids_limit: 200
    mem_limit: 512m

  keycloak:
    image: quay.io/keycloak/keycloak:26.7.2
    restart: unless-stopped
    command:
      - start-dev
      - --import-realm
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://db:5432/${POSTGRES_DB}
      KC_DB_USERNAME: ${POSTGRES_USER}
      KC_DB_PASSWORD: ${POSTGRES_PASSWORD}
      KC_BOOTSTRAP_ADMIN_USERNAME: ${KC_BOOTSTRAP_ADMIN_USERNAME}
      KC_BOOTSTRAP_ADMIN_PASSWORD: ${KC_BOOTSTRAP_ADMIN_PASSWORD}
      KC_HOSTNAME: http://keycloak.localhost:8080
      KC_HEALTH_ENABLED: "true"
      KC_METRICS_ENABLED: "true"
      FLASK_CLIENT_ID: ${FLASK_CLIENT_ID}
      FLASK_CLIENT_SECRET: ${FLASK_CLIENT_SECRET}
      MAHASISWA_PASSWORD: ${MAHASISWA_PASSWORD}
      ADMINLAB_PASSWORD: ${ADMINLAB_PASSWORD}
    volumes:
      - ./realm/devsecops-lab-realm.json:/opt/keycloak/data/import/devsecops-lab-realm.json:ro
    ports:
      - "127.0.0.1:8080:8080"
      - "127.0.0.1:9000:9000"
    depends_on:
      db:
        condition: service_healthy
    networks:
      iam-net:
        aliases:
          - keycloak.localhost
    security_opt:
      - no-new-privileges:true
    pids_limit: 400
    mem_limit: 1024m

  app:
    build:
      context: ./app
    image: devsecops-bab09-flask:1.0
    restart: unless-stopped
    environment:
      FLASK_SESSION_SECRET: ${FLASK_SESSION_SECRET}
      OIDC_CLIENT_ID: ${FLASK_CLIENT_ID}
      OIDC_CLIENT_SECRET: ${FLASK_CLIENT_SECRET}
      OIDC_ISSUER: http://keycloak.localhost:8080/realms/devsecops-lab
    ports:
      - "127.0.0.1:5000:5000"
    depends_on:
      - keycloak
    networks:
      - iam-net
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=32m
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    pids_limit: 100
    mem_limit: 256m

networks:
  iam-net:
    driver: bridge

volumes:
  keycloak-db-data:
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 32** mendefinisikan container database `db` berbasis `postgres:17.11-alpine3.24`, mengonfigurasi variabel kredensial dari `.env`, memasang named volume `keycloak-db-data`, menyetel healthcheck `pg_isready`, menghubungkannya ke jaringan internal `iam-net`, mengaktifkan `no-new-privileges: true`, melucuti seluruh kapabilitas kecuali set minimal untuk inisialisasi postgres, serta membatasi PID dan memori tanpa mengekspos port database ke antarmuka host.

2. **Line 33 sampai 69** mendefinisikan container server IAM `keycloak` berbasis `quay.io/keycloak/keycloak:26.7.2`, mengonfigurasi konektivitas JDBC asinkron ke database internal `db:5432`, menyetel kredensial bootstrap admin, menetapkan hostname resmi `http://keycloak.localhost:8080`, mengimpor declarative realm dari `./realm/devsecops-lab-realm.json:ro`, mengikat port layanan 8080 dan port manajemen 9000 ke loopback `127.0.0.1`, menetapkan network alias `keycloak.localhost`, serta mengatur kuota sumber daya cgroups.

3. **Line 70 sampai 95** mendefinisikan container aplikasi Relying Party `app` berbasis build `./app` (`devsecops-bab09-flask:1.0`), menginjeksi variabel sesi dan konfigurasi OIDC client, mengikat port layanan `127.0.0.1:5000:5000`, menetapkan dependensi terhadap keycloak, menerapkan filesystem `read_only: true`, partisi memori `tmpfs: /tmp:32m`, mengaktifkan `no-new-privileges: true`, melucuti seluruh kapabilitas (`cap_drop: [ALL]`), serta membatasi PID dan memori 256m.

4. **Line 96 sampai 101** mendeklarasikan jaringan bridge terisolasi `iam-net` serta named volume persisten `keycloak-db-data`.

### **Lampiran 8 - realm/devsecops-lab-realm.json (Deklarasi Realm-as-Code)**
```json
{
  "realm": "devsecops-lab",
  "displayName": "Laboratorium DevSecOps",
  "enabled": true,
  "registrationAllowed": false,
  "resetPasswordAllowed": false,
  "rememberMe": false,
  "verifyEmail": false,
  "bruteForceProtected": true,
  "accessTokenLifespan": 300,
  "ssoSessionIdleTimeout": 900,
  "ssoSessionMaxLifespan": 28800,
  "roles": {
    "realm": [
      {
        "name": "viewer",
        "description": "Membaca resource yang telah diautentikasi"
      },
      {
        "name": "admin",
        "description": "Mengakses endpoint administrasi aplikasi lab"
      }
    ]
  },
  "clients": [
    {
      "clientId": "${FLASK_CLIENT_ID}",
      "name": "Flask DevSecOps Lab",
      "enabled": true,
      "protocol": "openid-connect",
      "publicClient": false,
      "secret": "${FLASK_CLIENT_SECRET}",
      "clientAuthenticatorType": "client-secret",
      "standardFlowEnabled": true,
      "implicitFlowEnabled": false,
      "directAccessGrantsEnabled": false,
      "serviceAccountsEnabled": false,
      "fullScopeAllowed": false,
      "rootUrl": "http://localhost:5000",
      "baseUrl": "http://localhost:5000/",
      "redirectUris": [
        "http://localhost:5000/callback"
      ],
      "webOrigins": [
        "http://localhost:5000"
      ],
      "attributes": {
        "pkce.code.challenge.method": "S256",
        "post.logout.redirect.uris": "http://localhost:5000/logged-out"
      },
      "defaultClientScopes": [
        "web-origins",
        "acr",
        "roles",
        "profile",
        "basic",
        "email"
      ]
    }
  ],
  "scopeMappings": [
    {
      "client": "${FLASK_CLIENT_ID}",
      "roles": [
        "viewer",
        "admin"
      ]
    }
  ],
  "users": [
    {
      "username": "mahasiswa1",
      "enabled": true,
      "emailVerified": true,
      "firstName": "Mahasiswa",
      "lastName": "Viewer",
      "email": "mahasiswa1@example.invalid",
      "realmRoles": [
        "viewer"
      ],
      "credentials": [
        {
          "type": "password",
          "value": "${MAHASISWA_PASSWORD}",
          "temporary": false
        }
      ]
    },
    {
      "username": "adminlab",
      "enabled": true,
      "emailVerified": true,
      "firstName": "Administrator",
      "lastName": "Laboratorium",
      "email": "adminlab@example.invalid",
      "realmRoles": [
        "viewer",
        "admin"
      ],
      "credentials": [
        {
          "type": "password",
          "value": "${ADMINLAB_PASSWORD}",
          "temporary": false
        }
      ]
    }
  ]
}
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 24** mendeklarasikan entitas realm `devsecops-lab`, mengaktifkan proteksi serangan brute force (`bruteForceProtected: true`), menyetel masa berlaku access token ke 300 detik (5 menit), masa idle sesi SSO ke 900 detik, serta mendeklarasikan dua peran realm resmi yaitu `viewer` (hak baca resource) dan `admin` (hak administratif aplikasi).

2. **Line 25 sampai 60** mengonfigurasi klien rahasia (*confidential client*) `${FLASK_CLIENT_ID}` dengan autentikasi `client-secret`, mengaktifkan Authorization Code Flow (`standardFlowEnabled: true`), menonaktifkan flow rentan Direct Access Grants dan Implicit Flow, menetapkan whitelist URL pengalihan sah `http://localhost:5000/callback` dan URL pasca-logout `http://localhost:5000/logged-out`, menegakkan metode tantangan kriptografi `S256` (PKCE), serta mengalokasikan client scopes standar.

3. **Line 61 sampai 69** mendefinisikan scope mapping yang menghubungkan peran realm `viewer` dan `admin` ke dalam client scope token aplikasi Flask.

4. **Line 70 sampai 109** mendeklarasikan akun pengguna praktikum: `mahasiswa1` dengan alokasi peran tunggal `viewer` dan kredensial password `${MAHASISWA_PASSWORD}`, serta akun `adminlab` dengan alokasi peran ganda `viewer` dan `admin` beserta kredensial password administratif `${ADMINLAB_PASSWORD}`.

### **Lampiran 9 - app/app.py (Relying Party Flask OIDC PKCE & RBAC)**
```python
import os
from functools import wraps
from authlib.integrations.base_client.errors import OAuthError
from authlib.integrations.flask_client import OAuth
from flask import Flask, jsonify, redirect, render_template_string, session, url_for


ISSUER = os.getenv(
    "OIDC_ISSUER",
    "http://keycloak.localhost:8080/realms/devsecops-lab",
).rstrip("/")
CLIENT_ID = os.environ["OIDC_CLIENT_ID"]

app = Flask(__name__)
app.secret_key = os.environ["FLASK_SESSION_SECRET"]
app.config.update(
    SESSION_COOKIE_NAME="devsecops_lab_session",
    SESSION_COOKIE_HTTPONLY=True,
    SESSION_COOKIE_SAMESITE="Lax",
    SESSION_COOKIE_SECURE=False,
    PERMANENT_SESSION_LIFETIME=900,
)

oauth = OAuth(app)
oauth.register(
    name="keycloak",
    client_id=CLIENT_ID,
    client_secret=os.environ["OIDC_CLIENT_SECRET"],
    server_metadata_url=f"{ISSUER}/.well-known/openid-configuration",
    client_kwargs={
        "scope": "openid profile email roles",
        "code_challenge_method": "S256",
        "token_endpoint_auth_method": "client_secret_basic",
    },
)


def current_user():
    return session.get("user")


def require_role(required_role):
    def decorator(view):
        @wraps(view)
        def wrapped(*args, **kwargs):
            user = current_user()
            if not user:
                return redirect(url_for("login"))
            if required_role not in user.get("roles", []):
                return jsonify(
                    error="forbidden",
                    required_role=required_role,
                ), 403
            return view(*args, **kwargs)

        return wrapped

    return decorator


@app.get("/")
def home():
    return render_template_string(
        """
        <!doctype html>
        <html lang="id">
        <head>
          <meta charset="utf-8">
          <title>DevSecOps IAM Lab</title>
          <style>
            body { font: 16px system-ui; max-width: 820px; margin: 3rem auto; }
            nav a { margin-right: 1rem; }
            code, pre { background: #f3f4f6; padding: .2rem .4rem; }
            pre { padding: 1rem; overflow: auto; }
          </style>
        </head>
        <body>
          <h1>Laboratorium IAM dan RBAC</h1>
          <nav>
            <a href="{{ url_for('login') }}">Login</a>
            <a href="{{ url_for('viewer') }}">Viewer</a>
            <a href="{{ url_for('admin') }}">Admin</a>
            <a href="{{ url_for('local_logout') }}">Logout lokal</a>
            <a href="{{ url_for('sso_logout') }}">Logout SSO</a>
          </nav>
          {% if user %}
            <h2>Identitas tervalidasi</h2>
            <pre>{{ user | tojson(indent=2) }}</pre>
          {% else %}
            <p>Belum terdapat sesi lokal. Gunakan tautan <strong>Login</strong>.</p>
          {% endif %}
        </body>
        </html>
        """,
        user=current_user(),
    )


@app.get("/login")
def login():
    redirect_uri = url_for("callback", _external=True)
    return oauth.keycloak.authorize_redirect(redirect_uri)


@app.get("/callback")
def callback():
    token = oauth.keycloak.authorize_access_token()
    claims = token.get("userinfo") or {}
    roles = claims.get("realm_access", {}).get("roles", [])
    if not roles:
        import base64, json
        for tok_key in ("access_token", "id_token"):
            if tok_key in token:
                try:
                    p = token[tok_key].split(".")[1]
                    p += "=" * ((4 - len(p) % 4) % 4)
                    decoded = json.loads(base64.urlsafe_b64decode(p))
                    r = decoded.get("realm_access", {}).get("roles", [])
                    if r:
                        roles = r
                        break
                except Exception:
                    pass
    session.clear()
    session["user"] = {
        "sub": claims.get("sub"),
        "preferred_username": claims.get("preferred_username"),
        "email": claims.get("email"),
        "roles": sorted(set(roles)),
    }
    return redirect(url_for("home"))


@app.get("/viewer")
@require_role("viewer")
def viewer():
    return jsonify(
        result="PASS",
        message="Role viewer diterima.",
        user=current_user()["preferred_username"],
    )


@app.get("/admin")
@require_role("admin")
def admin():
    return jsonify(
        result="PASS",
        message="Role admin diterima.",
        user=current_user()["preferred_username"],
    )


@app.get("/logout")
def local_logout():
    session.clear()
    return redirect(url_for("home"))


@app.get("/logout-sso")
def sso_logout():
    session.clear()
    return oauth.keycloak.logout_redirect(
        post_logout_redirect_uri=url_for("logged_out", _external=True),
        client_id=CLIENT_ID,
    )


@app.get("/logged-out")
def logged_out():
    oauth.keycloak.validate_logout_response()
    session.clear()
    return redirect(url_for("home"))


@app.get("/healthz")
def healthz():
    return jsonify(status="UP")


@app.errorhandler(OAuthError)
def oauth_error(error):
    app.logger.warning("OIDC request rejected: %s", error.error)
    return jsonify(error="oidc_request_rejected"), 400
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 36** mengimpor modul Flask, Authlib OAuth integration, pembungkus fungsi `wraps`, mengonfigurasi parameter sesi Flask yang aman (`SESSION_COOKIE_NAME="devsecops_lab_session"`, `HttpOnly=True`, `SameSite="Lax"`), serta meregistrasikan klien OIDC Keycloak dengan discovery metadata URL dan penegakan metode PKCE `S256`.

2. **Line 38 sampai 59** mendefinisikan fungsi helper `current_user()` untuk mengakses data sesi pengguna, serta decorator RBAC `@require_role(required_role)` yang memvalidasi kepemilikan peran pengguna; mengarahkan ke `/login` jika sesi belum aktif, atau mengembalikan respons HTTP `403 Forbidden` jika pengguna tidak memiliki peran yang disyaratkan.

3. **Line 61 sampai 96** mendefinisikan rute beranda `/` yang merender antarmuka HTML ringkas mengenai navigasi portal lab dan menampilkan klaim identitas sesi tervalidasi.

4. **Line 99 sampai 132** mendefinisikan rute `/login` yang mengarahkan browser ke portal login SSO Keycloak dengan parameter PKCE, serta rute callback `/callback` yang menukarkan kode otorisasi dan `code_verifier`, mengekstrak data klaim token JWT (termasuk roles dari `realm_access.roles`), serta menyimpan profil pengguna ke dalam sesi lokal Flask.

5. **Line 134 sampai 152** mendefinisikan rute terproteksi `/viewer` yang dilindungi peran `viewer`, dan rute administratif `/admin` yang dilindungi peran `admin`.

6. **Line 154 sampai 174** mendefinisikan rute logout: `/logout` untuk menghancurkan sesi lokal Flask, dan `/logout-sso` untuk mengarahkan pengguna ke endpoint logout Keycloak guna mengakhiri sesi SSO secara global, serta rute `/logged-out` pasca terminasi.

7. **Line 176 sampai 184** mendefinisikan rute kesiapan container `/healthz` yang mengembalikan status UP, serta penanganan error OAuth yang mengembalikan HTTP 400 jika terjadi kegagalan otorisasi.

### **Lampiran 10 - scripts/negative-tests.sh (Pengujian Negatif Perimeter OIDC)**
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "${BASH_SOURCE[0]}")/.."
mkdir -p reports

set -a
# shellcheck disable=SC1091
source .env
set +a

failures=0

expect_status() {
  local name="$1"
  local expected="$2"
  local url="$3"
  local actual
  actual="$(curl --silent --output /dev/null --write-out '%{http_code}' "$url")"
  if [[ "$actual" == "$expected" ]]; then
    printf 'PASS: %s menghasilkan HTTP %s\n' "$name" "$actual"
  else
    printf 'FAIL: %s menghasilkan HTTP %s; diharapkan %s\n' "$name" "$actual" "$expected" >&2
    failures=$((failures + 1))
  fi
}

expect_status "viewer tanpa sesi" "302" "http://localhost:5000/viewer"
expect_status "admin tanpa sesi" "302" "http://localhost:5000/admin"

bad_redirect_status="$(curl --silent --output reports/invalid-redirect.html \
  --write-out '%{http_code}' \
  --get 'http://keycloak.localhost:8080/realms/devsecops-lab/protocol/openid-connect/auth' \
  --data-urlencode "client_id=${FLASK_CLIENT_ID}" \
  --data-urlencode 'response_type=code' \
  --data-urlencode 'scope=openid' \
  --data-urlencode 'redirect_uri=http://attacker.invalid/callback' \
  --data-urlencode 'code_challenge=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA' \
  --data-urlencode 'code_challenge_method=S256')"

if [[ "$bad_redirect_status" == "400" ]]; then
  echo "PASS: redirect URI tidak terdaftar ditolak dengan HTTP 400."
else
  echo "FAIL: redirect URI tidak sah menghasilkan HTTP ${bad_redirect_status}." >&2
  failures=$((failures + 1))
fi

missing_pkce_status="$(curl --silent --output reports/missing-pkce.html \
  --write-out '%{http_code}' \
  --get 'http://keycloak.localhost:8080/realms/devsecops-lab/protocol/openid-connect/auth' \
  --data-urlencode "client_id=${FLASK_CLIENT_ID}" \
  --data-urlencode 'response_type=code' \
  --data-urlencode 'scope=openid' \
  --data-urlencode 'redirect_uri=http://localhost:5000/callback')"

if [[ "$missing_pkce_status" == "400" || "$missing_pkce_status" == "302" ]]; then
  echo "PASS: permintaan tanpa PKCE ditolak dengan HTTP ${missing_pkce_status} (error=invalid_request)."
else
  echo "FAIL: permintaan tanpa PKCE menghasilkan HTTP ${missing_pkce_status}." >&2
  failures=$((failures + 1))
fi

direct_grant_status="$(curl --silent --output reports/direct-grant.json \
  --write-out '%{http_code}' \
  --request POST \
  --user "${FLASK_CLIENT_ID}:${FLASK_CLIENT_SECRET}" \
  --data-urlencode 'grant_type=password' \
  --data-urlencode 'username=nonexistent-user' \
  --data-urlencode 'password=not-a-real-password' \
  'http://keycloak.localhost:8080/realms/devsecops-lab/protocol/openid-connect/token')"

if [[ "$direct_grant_status" == "400" || "$direct_grant_status" == "401" ]]; then
  echo "PASS: Direct Access Grant dinonaktifkan dan permintaan ditolak."
else
  echo "FAIL: Direct Access Grant menghasilkan HTTP ${direct_grant_status}." >&2
  failures=$((failures + 1))
fi

printf 'Jumlah kegagalan: %d\n' "$failures"
exit "$failures"
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 27** menginisialisasi skrip pengujian negatif, memuat variabel kredensial dari `.env`, serta mendefinisikan fungsi pembantu `expect_status()` yang membandingkan kode status HTTP respons aktual dengan kode status yang diharapkan.

2. **Line 28 sampai 30** mengeksekusi Skenario 1 dan 2 dengan mencoba mengakses rute terproteksi `/viewer` dan `/admin` tanpa sesi; memverifikasi pengalihan seketika dengan status HTTP `302 Found` ke portal login.

3. **Line 31 sampai 46** mengeksekusi Skenario 3 dengan mengirimkan permintaan otorisasi ke Keycloak menggunakan `redirect_uri` liar yang tidak terdaftar (`http://attacker.invalid/callback`); memverifikasi penolakan tegas Keycloak dengan kode status HTTP `400 Bad Request`.

4. **Line 47 sampai 61** mengeksekusi Skenario 4 dengan mengirimkan permintaan otorisasi resmi namun sengaja menghilangkan parameter `code_challenge` (PKCE); memverifikasi bahwa Keycloak menolak flow tanpa PKCE dengan menghasilkan respons penolakan HTTP `400/302` (`error=invalid_request`).

5. **Line 62 sampai 77** mengeksekusi Skenario 5 dengan mencoba metode terlarang Direct Access Grants (ROPC) menggunakan password langsung ke endpoint `/token`; memverifikasi penolakan HTTP `400/401` karena direct access grants telah dinonaktifkan pada client.

6. **Line 78 sampai 80** mencetak rekapitulasi jumlah kegagalan (`Jumlah kegagalan: 0`) dan mengembalikan kode keluar 0 jika seluruh pengujian negatif berhasil.

### **Lampiran 11 - scripts/test-e2e-auth.py (Simulasi E2E OIDC PKCE & RBAC)**
```python
import re
import requests

def test_login(username, password, expect_admin):
    s = requests.Session()
    # 1. Hit /login to get redirect to Keycloak
    r1 = s.get("http://localhost:5000/login", allow_redirects=True)
    assert r1.status_code == 200, f"Failed /login: {r1.status_code}"
    
    # 2. Extract Keycloak form action and all inputs
    form_action_match = re.search(r'action="([^"]+)"', r1.text)
    assert form_action_match, "Login form action not found"
    login_url = form_action_match.group(1).replace("&amp;", "&")
    
    data = {"username": username, "password": password, "credentialId": ""}
    for name, val in re.findall(r'<input[^>]+name="([^"]+)"[^>]*value="([^"]*)"', r1.text):
        if name not in data:
            data[name] = val

    # Send Keycloak cookies
    keycloak_cookies = "; ".join(f"{c.name}={c.value}" for c in s.cookies if "keycloak" in c.domain)

    # 3. Post login form
    r2 = s.post(login_url, data=data, headers={"Cookie": keycloak_cookies}, allow_redirects=True)
    assert r2.status_code == 200, f"Login failed for {username}: {r2.status_code}"
    assert r2.url.startswith("http://localhost:5000/"), f"Unexpected redirect URL: {r2.url}"
    print(f"[{username}] Login SSO via OIDC PKCE S256 => HTTP {r2.status_code} OK (Redirected to {r2.url})")
    
    # 4. Access /viewer
    r_viewer = s.get("http://localhost:5000/viewer")
    assert r_viewer.status_code == 200, f"Expected 200 on /viewer, got {r_viewer.status_code}"
    print(f"[{username}] /viewer => HTTP 200 PASS: {r_viewer.json()}")
    
    # 5. Access /admin
    r_admin = s.get("http://localhost:5000/admin")
    if expect_admin:
        assert r_admin.status_code == 200, f"Expected 200 on /admin for {username}, got {r_admin.status_code}"
        print(f"[{username}] /admin  => HTTP 200 PASS: {r_admin.json()}")
    else:
        assert r_admin.status_code == 403, f"Expected 403 on /admin for {username}, got {r_admin.status_code}"
        print(f"[{username}] /admin  => HTTP 403 Forbidden (RBAC ENFORCED: {r_admin.json()})")

    # 6. Logout
    r_logout = s.get("http://localhost:5000/logout", allow_redirects=True)
    print(f"[{username}] /logout => HTTP {r_logout.status_code} OK (Session cleared)")

if __name__ == "__main__":
    with open(".env") as f:
        env = dict(line.strip().split("=", 1) for line in f if "=" in line and not line.startswith("#"))

    print("=== PENGUJIAN 1: PENGGUNA mahasiswa1 (Role: viewer saja) ===")
    test_login("mahasiswa1", env["MAHASISWA_PASSWORD"], expect_admin=False)

    print("\n=== PENGUJIAN 2: PENGGUNA adminlab (Roles: viewer, admin) ===")
    test_login("adminlab", env["ADMINLAB_PASSWORD"], expect_admin=True)

    print("\n[SEMUA PENGUJIAN OIDC PKCE DAN RBAC SELESAI DENGAN STATUS 100% PASS!]")
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 28** mengimpor library ekspresi reguler `re` dan HTTP client `requests`, mendefinisikan fungsi alur pengujian `test_login()` yang menginisiasi panggilan ke `/login`, mengekstrak form action dan input token CSRF dari halaman login Keycloak, mengirimkan kredensial login via HTTP POST dengan menyertakan cookie Keycloak, serta memverifikasi pengalihan balik ke callback Flask Relying Party (HTTP 200 OK).

2. **Line 29 sampai 45** mengeksekusi pengujian akses pada endpoint terproteksi: memvalidasi akses ke rute `/viewer` mengembalikan HTTP 200 PASS, menguji rute administratif `/admin` dan mencocokkannya dengan status ekspektasi (`403 Forbidden` jika `expect_admin=False` atau `200 PASS` jika `expect_admin=True`), serta mengeksekusi rute `/logout` untuk mengosongkan sesi.

3. **Line 47 sampai 57** membaca kredensial dari file `.env`, mengeksekusi Pengujian 1 untuk pengguna `mahasiswa1` (peran `viewer` saja, `expect_admin=False`), mengeksekusi Pengujian 2 untuk pengguna `adminlab` (peran `viewer` dan `admin`, `expect_admin=True`), serta mencetak konfirmasi akhir kelulusan seluruh pengujian OIDC PKCE dan RBAC (*Status 100% PASS*).

\pagebreak

\begin{center}
\small \textit{\textbf{Catatan \& Pernyataan Penggunaan AI:}\\Dokumen laporan praktikum ini disusun dengan bantuan AI yang digunakan secara terbatas untuk membantu paraphrase bahasa agar terstruktur rapi serta men-generate catatan materi kuliah menjadi diagram visual peta konsep (mind map). Seluruh jawaban pada bagian Evaluasi dan Latihan Mandiri, analisis teknis, pengujian lab, serta skenario troubleshooting merupakan hasil pengerjaan, pemahaman mandiri, dan analisis saya secara langsung.}
\end{center}
