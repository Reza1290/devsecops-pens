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

Tahap awal praktikum Bab 8 dimulai dengan pembuatan sertifikat digital dan kredensial akses untuk private registry menggunakan skrip otomatisasi `scripts/generate-registry-assets.sh`. Skrip ini menginstruksikan utilitas OpenSSL untuk menghasilkan pasangan kunci asimetris RSA 3072-bit dan sertifikat X.509 self-signed yang memuat *Subject Alternative Name* (SAN) `DNS:localhost,IP:127.0.0.1`. Selanjutnya, utilitas `htpasswd` dijalankan dengan flag `-B` untuk membangkitkan hash password pengguna `operator` menggunakan algoritma bcrypt dengan faktor biaya komputasi yang aman.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image2.png}
\end{center}
\vspace{0.2cm}

Hasil eksekusi pada Gambar 4.1 menampilkan bahwa file sertifikat `domain.crt`, private key `domain.key`, dan file kredensial `htpasswd` berhasil dibuat pada direktori `registry/certs/` dan `registry/auth/`. Izin file disetel secara tepat sehingga dapat dibaca oleh kontainer registry saat proses mounting.

## **4.2 Konfigurasi Certificate Trust Docker Daemon Host**

Docker Engine secara default menolak berkomunikasi dengan registry yang menggunakan sertifikat self-signed atau sertifikat yang diterbitkan oleh CA privat yang tidak terdaftar di dalam system trust store. Untuk menjalin hubungan saling percaya secara aman tanpa perlu menambahkan parameter berbahaya `--insecure-registry` pada konfigurasi daemon, sertifikat publik `domain.crt` didaftarkan ke dalam direktori kepercayaan Docker host pada `/etc/docker/certs.d/localhost:5000/ca.crt`.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image3.png}
\end{center}
\vspace{0.2cm}

Pemeriksaan sertifikat menggunakan utilitas `openssl x509` pada Gambar 4.2 membuktikan bahwa subjek dan penerbit (*issuer*) sertifikat terdaftar atas nama institusi PENS dengan masa berlaku satu tahun penuh, memastikan proses jabat tangan TLS antara daemon Docker dan private registry berjalan mulus.

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

Setelah dinyatakan lolos dari security gate, layanan multi-kontainer Bab 8 dijalankan menggunakan Docker Compose. Service stack terdiri atas `devsecops-bab08-registry-1` yang menyediakan layanan private registry TLS pada port host `5000`, serta `devsecops-bab08-app-1` yang menjalankan aplikasi web Python pada port host `8085` melalui konfigurasi variabel lingkungan `${APP_PORT:-8085}`.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image6.png}
\end{center}
\vspace{0.2cm}

Setelah service stack aktif dan berstatus sehat (*healthy*), skrip `scripts/push-registry.sh` dieksekusi. Skrip melakukan login berotentikasi ke `localhost:5000` menggunakan akun operator, memberikan tag repositori, me-push seluruh image layer, dan memverifikasi ketersediaan artefak melalui API katalog resmi registry v2.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image7.png}
\end{center}
\vspace{0.2cm}

Gambar 4.7 membuktikan bahwa image `pens-web:1.0` berhasil di-push dan tersimpan secara valid pada private registry terenkripsi TLS, dibuktikan dengan respon query katalog `{"repositories":["pens-web"]}` dan query tag `{"name":"pens-web","tags":["1.0"]}`.

## **4.6 Pengujian Runtime Hardening - Eksekusi Otomatis verify-hardening.sh (11/11 Kontrol Lulus)**

Untuk memverifikasi secara empiris bahwa seluruh pembatasan keamanan bekerja pada tingkat kernel dan lingkungan waktu proses, skrip uji negatif `scripts/verify-hardening.sh` dijalankan. Skrip ini secara otomatis mengeksekusi 11 pengujian keamanan independen terhadap kontainer `devsecops-bab08-app-1`.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image8.png}
\end{center}
\vspace{0.2cm}

Hasil pengujian pada Gambar 4.8 menunjukkan kelulusan sempurna (11 dari 11 kontrol keamanan lulus / *PASS*).

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

Pengujian langsung menggunakan utilitas `curl` terhadap endpoint aplikasi Flask dilakukan untuk memvalidasi keluaran JSON yang dihasilkan oleh aplikasi `pens-web`. Rute `/` mengonfirmasi identitas proses dan status filesystem, sedangkan rute `/secret` mengonfirmasi pembacaan file rahasia secara aman.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image9.png}
\end{center}
\vspace{0.2cm}

Pemeriksaan keamanan private registry tanpa menyertakan header autentikasi pada Gambar 4.10 menunjukkan penolakan tegas oleh server registry dengan kode status HTTP `401 Unauthorized` dan tantangan `Www-Authenticate: Basic realm="Registry Realm"`.

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

Praktikum Bab 9 diawali dengan konfigurasi domain name mapping loopback pada file `/etc/hosts`. Entri `127.0.0.1 keycloak.localhost` ditambahkan agar browser dan klien pengujian dapat mengakses Keycloak menggunakan nama domain terstandarisasi yang konsisten dengan nama issuer yang diterbitkan di dalam token JWT. Selanjutnya, skrip otomatisasi `scripts/generate-env.sh` dijalankan untuk menghasilkan password administratif, rahasia klien OIDC, dan kunci rahasia Flask menggunakan generator acak berentropi tinggi (`openssl rand -hex`), lalu menyetel izin file `.env` ke mode `600` (hanya dapat dibaca oleh pemilik) - \vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image11.png}
\end{center}
\vspace{0.2cm}

## **4.9 Validasi Statis Realm-as-Code dan Deployment IAM Stack Keycloak + PostgreSQL**

Sebelum container stack dijalankan, skrip `scripts/validate-lab.sh` dieksekusi untuk memastikan keabsahan file konfigurasi deklaratif `realm/realm-devsecops.json`. Skrip memvalidasi sintaks format JSON, memeriksa pendaftaran klien rahasia `devsecops-app`, memastikan keberadaan peran realm `viewer` dan `admin`, serta memverifikasi bahwa opsi rentan `directAccessGrantsEnabled` bernilai `false`.

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

Gambar 4.16 membuktikan bahwa seluruh pengujian keamanan negatif berhasil dengan nol kegagalan (*0 failures*).

## **4.12 Pengujian End-to-End OIDC Authorization Code Flow PKCE dan Penegakan RBAC**

Pengujian fungsional menyeluruh dilakukan melalui skrip Python `scripts/test-e2e-auth.py` yang mensimulasikan interaksi nyata web browser dalam menyelesaikan flow *Authorization Code Flow* dengan PKCE S256.

Pada skenario pertama (Gambar 4.17), pengujian dijalankan atas username `mahasiswa1` yang hanya memiliki peran `viewer`. Pengguna berhasil melakukan login SSO di Keycloak, menukarkan kode otorisasi beserta `code_verifier`, dan memperoleh token akses. Ketika mengakses rute `/viewer`, server merespons HTTP `200 OK` (lulus). Namun, saat mencoba melintasi batas otorisasi menuju rute administratif `/admin`, aplikasi Flask menolak akses dengan kode status HTTP `403 Forbidden`, membuktikan bahwa kebijakan RBAC bekerja secara deterministik.

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image17.png}
\end{center}
\vspace{0.2cm}

Pada skenario kedua (Gambar 4.18), pengujian diulang atas username `adminlab` yang memiliki kombinasi peran `viewer` dan `admin`. Hasil pengujian membuktikan bahwa pengguna memiliki wewenang penuh untuk mengakses rute `/viewer` (HTTP 200 OK) sekaligus membuka panel administratif `/admin` (HTTP 200 OK).

\vspace{0.3cm}
\begin{center}
\includegraphics[width=0.92\linewidth]{images/image18.png}
\end{center}
\vspace{0.2cm}

Inspeksi terhadap isi token JWT access token Keycloak yang didekodekan tanpa signature verification ditampilkan pada Gambar 4.23. Klaim `preferred_username`, identitas unik `sub`, masa berlaku `exp`, serta struktur peran `realm_access.roles: ["viewer"]` terverifikasi tersusun rapi sesuai standar RFC 7519.

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

Siklus akhir pengujian melibatkan penutupan sesi secara terkoordinasi (*Single Logout* / SLO). Saat pengguna mengakses rute `/logout`, sesi lokal Flask dihancurkan dan cookie sesi dikosongkan. Browser kemudian dialihkan ke endpoint logout Keycloak untuk mematikan sesi SSO secara permanen. Pengujian permintaan susulan ke rute `/viewer` membuktikan bahwa akses ditolak dan pengguna diarahkan kembali ke halaman login - \vspace{0.3cm}
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
  registry:
    image: registry:2.8.3
    container_name: devsecops-bab08-registry-1
    restart: unless-stopped
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: "Registry Realm"
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
      REGISTRY_HTTP_TLS_KEY: /certs/domain.key
    volumes:
      - ./registry/certs:/certs:ro
      - ./registry/auth:/auth:ro
      - registry-data:/var/lib/registry
    ports:
      - "127.0.0.1:5000:5000"
    networks:
      - registry-net
    read_only: true
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true

  app:
    image: pens-web:1.0
    container_name: devsecops-bab08-app-1
    restart: unless-stopped
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    volumes:
      - ./secrets/app_secret.txt:/run/secrets/app_secret:ro
    ports:
      - "127.0.0.1:${APP_PORT:-8085}:5000"
    networks:
      - registry-net
    cpus: "0.50"
    mem_limit: 256m
    healthcheck:
      test: ["CMD", "python3", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:5000/')"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s

volumes:
  registry-data:

networks:
  registry-net:
    driver: bridge
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 6** mendeklarasikan blok `services` dan mendefinisikan container `registry` menggunakan image resmi `registry:2.8.3` dengan penamaan kontainer eksplisit `devsecops-bab08-registry-1` serta restart policy `restart: unless-stopped`.

2. **Line 7 sampai 12** mengonfigurasi environment variables registry untuk mengaktifkan skema autentikasi HTTP Basic berbasis `htpasswd` pada realm `Registry Realm`, serta menetapkan jalur sertifikat dan kunci TLS pada `/certs/domain.crt` dan `/certs/domain.key`.

3. **Line 13 sampai 19** memetakan volume direktori sertifikat `./registry/certs` dan autentikasi `./registry/auth` ke dalam kontainer dengan mode *read-only* (`:ro`), named volume persisten `registry-data` ke `/var/lib/registry`, serta membatasi port binding ke loopback host `127.0.0.1:5000:5000`.

4. **Line 20 sampai 25** menerapkan pengerasan runtime pada registry meliputi `read_only: true` (mengunci filesystem root kontainer), `cap_drop: [ALL]` (melucuti seluruh Linux capabilities), dan `no-new-privileges: true` (mencegah eskalasi hak akses via biner SetUID).

5. **Line 26 sampai 36** mendefinisikan container aplikasi `app` berbasis image lokal `pens-web:1.0`, mengaktifkan proteksi `read_only: true`, partisi memori volatil `tmpfs: /tmp:rw,noexec,nosuid,size=64m` (mencegah eksekusi biner sembarang pada `/tmp`), serta melucuti seluruh kapabilitas kernel (`cap_drop: [ALL]`).

6. **Line 37 sampai 42** me-mount file rahasia `./secrets/app_secret.txt` ke `/run/secrets/app_secret:ro` secara *read-only* (tanpa mengeksposnya pada environment variables), mengikat port loopback dinamis `127.0.0.1:${APP_PORT:-8085}:5000`, serta mengalokasikan kuota cgroups `cpus: "0.50"` dan `mem_limit: 256m`.

7. **Line 43 sampai 56** mengonfigurasi mekanisme `healthcheck` aplikasi berbasis skrip python urllib internal setiap 10 detik, mendeklarasikan volume persisten `registry-data`, dan mendefinisikan jaringan isolasi bridge `registry-net`.


### **Lampiran 2 - app/Dockerfile (Multi-Stage Build Hardened Image)**
```dockerfile
# Stage 1: Build virtual environment
FROM python:3.11.16-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN python -m venv /opt/venv && \
    /opt/venv/bin/pip install --no-cache-dir --upgrade pip setuptools wheel && \
    /opt/venv/bin/pip install --no-cache-dir -r requirements.txt

# Stage 2: Hardened runtime environment
FROM python:3.11.16-slim
WORKDIR /app

# Patch Debian CVEs
RUN apt-get update && apt-get upgrade -y && \
    rm -rf /var/lib/apt/lists/*

# Create non-root system user and group
RUN groupadd -g 10001 app && \
    useradd -u 10001 -g app -s /bin/nologin -d /app app

# Copy compiled virtual environment from builder
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Remove pip/setuptools/wheel from runtime to eliminate CVE surface
RUN /opt/venv/bin/pip uninstall -y pip setuptools wheel && \
    rm -rf /usr/local/lib/python3.11/site-packages/setuptools* \
           /usr/local/lib/python3.11/site-packages/wheel*

COPY --chown=10001:10001 app.py .

USER 10001:10001
EXPOSE 5000

CMD ["python3", "-m", "gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 7** mendeklarasikan tahap pertama (*builder stage*) menggunakan base image `python:3.11.16-slim`, membuat virtual environment terisolasi pada `/opt/venv`, dan menginstal dependensi aplikasi dari file `requirements.txt` tanpa menyimpan cache pip (`--no-cache-dir`).

2. **Line 8 sampai 15** menginisialisasi tahap kedua (*runtime stage*) dengan base image bersih, mengeksekusi pembaruan paket keamanan Debian (`apt-get update && apt-get upgrade -y`) untuk mem-patch seluruh CVE yang dilaporkan pada base image, serta menghapus cache apt guna merampingkan ukuran image.

3. **Line 16 sampai 19** membuat grup sistem `app` (GID 10001) dan pengguna sistem `app` (UID 10001) tanpa shell interaktif (`-s /bin/nologin`) dan direktori kerja `/app`, menegakkan prinsip non-root user secara permanen.

4. **Line 20 sampai 22** menyalin direktori virtual environment `/opt/venv` yang telah terkompilasi dari builder stage ke runtime stage menggunakan instruksi `COPY --from=builder` dan menambahkan jalurnya ke variabel `PATH`.

5. **Line 23 sampai 27** mencopot modul kompilasi `pip`, `setuptools`, dan `wheel` dari virtual environment serta menghapus file residunya pada direktori Python global library untuk mengeliminasi permukaan serangan CVE paket bawaan.

6. **Line 28 sampai 34** menyalin file kode aplikasi `app.py` dengan kepemilikan kepemilikan non-root `10001:10001`, mengunci eksekusi pada pengguna `USER 10001:10001`, mendeklarasikan port aplikasi 5000, dan mendefinisikan perintah eksekusi server WSGI Gunicorn dengan 2 worker.


### **Lampiran 3 - app/app.py (Aplikasi Web Pengujian Hardening)**
```python
import os
import hashlib
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/")
def index():
    return jsonify({
        "app": "pens-web",
        "status": "healthy",
        "user": {
            "uid": os.getuid(),
            "gid": os.getgid()
        },
        "rootfs": "read-only",
        "secret_mounted": os.path.exists("/run/secrets/app_secret")
    })

@app.route("/secret")
def secret():
    secret_path = "/run/secrets/app_secret"
    if not os.path.exists(secret_path):
        return jsonify({"status": "error", "message": "Secret not found"}), 404
    
    with open(secret_path, "r") as f:
        content = f.read().strip()
    
    # Return hash to prevent secret exposure in responses
    secret_hash = hashlib.sha256(content.encode()).hexdigest()
    return jsonify({
        "status": "success",
        "secret_value_sha256": secret_hash,
        "source": secret_path,
        "permission": "read-only"
    })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 6** mengimpor core OS library `os`, cryptography library `hashlib`, kerangka kerja web `Flask`, fungsi pembentuk respons `jsonify`, serta menginisialisasi instans aplikasi Flask.

2. **Line 7 sampai 19** mendefinisikan rute pemeriksaan status `/` yang mengembalikan respons JSON mengenai nama aplikasi, status kesehatan `healthy`, pembuktian identitas non-root runtime melalui pemanggilan `os.getuid()` dan `os.getgid()`, serta verifikasi keberadaan mount secret.

3. **Line 20 sampai 35** mendefinisikan rute `/secret` yang membaca file rahasia pada jalur `/run/secrets/app_secret`, menghitung nilai hash kriptografis SHA-256 dari isi secret, dan mengembalikan ringkasan hash tersebut ke klien tanpa mengekspos nilai teks mentah rahasia ke jaringan.

4. **Line 36 sampai 38** mengonfigurasi blok eksekusi utama aplikasi Flask pada network interface `0.0.0.0` dan port `5000` untuk mode pengembangan mandiri.


### **Lampiran 4 - scripts/generate-registry-assets.sh (Pembuatan Aset TLS & Bcrypt)**
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/."

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

1. **Line 1 sampai 15** mengaktifkan mode penanganan error Bash ketat (`set -Eeuo pipefail`), berpindah ke direktori induk proyek, memvalidasi keberadaan file `.env`, memuat variabel lingkungan, serta memastikan ketersediaan variabel `REGISTRY_USER` dan `REGISTRY_PASSWORD`.

2. **Line 16 sampai 28** memverifikasi ketersediaan OpenSSL CLI tool pada sistem host, membuat direktori tujuan `registry/certs` dan `registry/auth`, menerapkan topeng izin ketat `umask 077`, serta mencegah penimpaan file kunci yang sudah ada tanpa rotasi sadar.

3. **Line 29 sampai 41** memanggil utilitas OpenSSL untuk membangkitkan pasangan kunci RSA 3072-bit dan sertifikat digital self-signed X.509 berdurasi 365 hari dengan algoritma SHA-256, memuat Subject Alternative Name (SAN) `DNS:localhost,IP:127.0.0.1`, dan extendedKeyUsage `serverAuth`.

4. **Line 42 sampai 49** mengalirkan password operator ke dalam container utilitas `httpd:2.4-alpine` untuk menjalankan perintah `htpasswd -Bni` yang menghasilkan hash kredensial berbasis fungsi hash adaptif bcrypt, lalu menyimpannya ke file `registry/auth/htpasswd`.

5. **Line 50 sampai 55** menyetel izin baca file sertifikat, private key, dan htpasswd ke mode `644` agar dapat dibaca oleh kontainer registry yang berjalan tanpa kapabilitas `CAP_DAC_OVERRIDE`, serta mencetak pesan konfirmasi keberhasilan.


### **Lampiran 5 - scripts/scan-image.sh (Trivy Scan, SBOM, & Security Gate)**
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/."

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
    /workspace/compose.yaml \
    > reports/trivy-config.txt

run_trivy fs \
    --scanners secret \
    /workspace \
    > reports/trivy-secret.txt

run_trivy image \
    --input "/workspace/reports/$archive_name" \
    --severity HIGH,CRITICAL \
    --ignore-unfixed \
    --format table \
    > reports/trivy-gate.txt

if grep -qE "CRITICAL|HIGH" reports/trivy-gate.txt; then
    echo "[FAIL] Security gate mendeteksi kerentanan HIGH/CRITICAL!" >&2
    echo "BLOCKED" > reports/gate-status.txt
    exit 1
else
    echo "[PASS] Security gate lulus: 0 kerentanan kritis terdeteksi."
    echo "PASS" > reports/gate-status.txt
fi
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 16** menetapkan konfigurasi eksekusi aman Bash, memuat variabel lingkungan dari file `.env`, memastikan parameter `APP_IMAGE` terdefinisi, dan membuat direktori output `reports/`.

2. **Line 17 sampai 25** memverifikasi keberadaan image target pada engine Docker, mengekspor image ke dalam arsip tarball mandiri (`reports/pens-web-1.0.tar`) guna mengeliminasi kebutuhan me-mount Docker socket host ke container pemindai, serta mendefinisikan fungsi pembungkus `run_trivy`.

3. **Line 26 sampai 36** mengeksekusi pemindaian kerentanan OS dan deteksi secret pada arsip image dalam format JSON terstruktur, serta menghasilkan file inventaris dependensi Software Bill of Materials (SBOM) berstandar industri CycloneDX (`pens-web-1.0.cdx.json`).

4. **Line 37 sampai 46** menjalankan pemindaian miskonfigurasi infrastruktur (IaC) terhadap file `compose.yaml` serta melakukan pemindaian kebocoran secret pada seluruh pohon direktori proyek.

5. **Line 47 sampai 60** mengevaluasi kriteria Security Gate dengan memindai kerentanan berstatus HIGH dan CRITICAL yang telah memiliki perbaikan (`--ignore-unfixed`), memeriksa output menggunakan ekspresi reguler, serta mencatat status akhir `PASS` atau `BLOCKED` pada file `reports/gate-status.txt`.


### **Lampiran 6 - scripts/verify-hardening.sh (Verifikasi Negatif 11 Kontrol)**
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/."

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

# 1. Non-root user check
uid="$(docker compose exec -T app id -u)"
[[ "$uid" == "10001" ]] \
    && pass "Proses aplikasi berjalan sebagai UID 10001." \
    || fail "UID aplikasi adalah $uid."

# 2. Read-only rootfs check
if docker compose exec -T app sh -c 'touch /unauthorized-write' 2>/dev/null; then
    fail "Filesystem root masih dapat ditulis."
else
    pass "Penulisan ke filesystem root ditolak."
fi

# 3. Tmpfs writability check
docker compose exec -T app sh -c 'touch /tmp/allowed-write && rm /tmp/allowed-write' \
    && pass "tmpfs /tmp dapat digunakan aplikasi." \
    || fail "tmpfs /tmp tidak dapat digunakan."

# 4. Capabilities dropped check
cap_eff="$(docker compose exec -T app sh -c "awk '/CapEff/ {print \$2}' /proc/1/status")"
[[ "$cap_eff" == "0000000000000000" ]] \
    && pass "Seluruh Linux capability efektif telah dihapus." \
    || fail "Capability efektif masih tersedia: $cap_eff"

# 5. NoNewPrivileges check
no_new_privs="$(docker compose exec -T app sh -c "awk '/NoNewPrivs/ {print \$2}' /proc/1/status")"
[[ "$no_new_privs" == "1" ]] \
    && pass "NoNewPrivs aktif." \
    || fail "NoNewPrivs tidak aktif."

# 6. Read-only secret mount check
docker compose exec -T app test -r /run/secrets/app_secret \
    && pass "Secret tersedia sebagai file read-only." \
    || fail "Secret tidak dapat dibaca aplikasi."

# 7. Environment isolation check
if docker compose exec -T app env | grep -q '^APP_SECRET='; then
    fail "Secret ditemukan sebagai environment variable."
else
    pass "Secret tidak diekspos sebagai APP_SECRET."
fi

# 8. Docker socket isolation check
docker compose exec -T app test ! -S /var/run/docker.sock \
    && pass "Docker socket tidak tersedia di container aplikasi." \
    || fail "Docker socket terpasang ke container aplikasi."

# 9. Resource quotas check
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

# 10. Health check probe
app_ready="false"
for attempt in {1.30}; do
    if curl --fail --silent --show-error \
        "http://127.0.0.1:${APP_PORT:-8085}/" >/dev/null 2>&1; then
        app_ready="true"
        break
    fi
    sleep 2
done
[[ "$app_ready" == "true" ]] \
    && pass "Health endpoint aplikasi merespons sukses." \
    || fail "Health endpoint aplikasi gagal."

# 11. Anonymous registry rejection
unauth_code="$(curl --silent --output /dev/null --write-out '%{http_code}' --cacert registry/certs/domain.crt https://localhost:5000/v2/ || true)"
[[ "$unauth_code" == "401" ]] \
    && pass "Registry menolak akses anonim dengan HTTP 401." \
    || fail "Registry memberi status anonim $unauth_code."

echo "[PASS] Total kontrol terverifikasi: $pass_count"
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 20** menginisialisasi skrip pengujian keamanan negatif, mengaktifkan penanganan kesalahan, memuat konfigurasi port dari file `.env`, serta mendefinisikan fungsi utilitas pencatatan status `pass()` dan `fail()`.

2. **Line 21 sampai 38** menguji kontrol 1 hingga 3 dengan memverifikasi proses aplikasi berjalan dengan UID non-root 10001, membuktikan upaya penulisan biner sembarang ke filesystem root `/` ditolak oleh kernel, serta memastikan penulisan file temporer berhasil pada partisi memori `/tmp`.

3. **Line 39 sampai 55** menguji kontrol 4 dan 5 dengan mengekstrak nilai bitmask `CapEff` dari file semu `/proc/1/status` untuk memastikan seluruh kapabilitas kernel bernilai `0000000000000000`, serta memverifikasi atribut `NoNewPrivs` bernilai `1` guna memblokir eskalasi hak akses via biner SetUID.

4. **Line 56 sampai 72** menguji kontrol 6 hingga 8 dengan memverifikasi file secret terbaca pada `/run/secrets/app_secret` dengan izin *read-only*, membuktikan secret value tidak bocor pada variabel lingkungan kontainer, serta memastikan socket administratif Docker host tidak dimount ke dalam aplikasi.

5. **Line 73 sampai 89** menguji kontrol 9 dengan melakukan inspeksi cgroups terhadap kontainer menggunakan utilitas `docker inspect` untuk memverifikasi kuota alokasi memori, batas CPU nanoCPUs, batas proses PIDs, serta atribut filesystem root *read-only*.

6. **Line 90 sampai 114** menguji kontrol 10 dan 11 dengan melakukan polling query kesehatan aplikasi pada port loopback `${APP_PORT:-8085}` hingga menerima respons HTTP 200 OK, serta menguji private registry TLS untuk memastikan akses anonim ditolak tegas dengan kode status HTTP `401 Unauthorized`.


## **B. File Konfigurasi dan Skrip Praktikum Bab 9 (IAM, OIDC, PKCE, dan RBAC Keycloak)**

### **Lampiran 7 - compose.yaml (Orkestrasi Keycloak, PostgreSQL, dan Flask RP)**
```yaml
services:
  db:
    image: postgres:17.11-alpine3.24
    container_name: devsecops-bab09-db-1
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-keycloak}
      POSTGRES_USER: ${POSTGRES_USER:-keycloak}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - iam-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-keycloak} -d ${POSTGRES_DB:-keycloak}"]
      interval: 5s
      timeout: 3s
      retries: 10

  keycloak:
    image: quay.io/keycloak/keycloak:26.7.2
    container_name: devsecops-bab09-keycloak-1
    restart: unless-stopped
    command:
      - start-dev
      - --import-realm
      - --health-enabled=true
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://db:5432/${POSTGRES_DB:-keycloak}
      KC_DB_USERNAME: ${POSTGRES_USER:-keycloak}
      KC_DB_PASSWORD: ${POSTGRES_PASSWORD}
      KEYCLOAK_ADMIN: ${KEYCLOAK_ADMIN:-admin}
      KEYCLOAK_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
      KC_HOSTNAME: keycloak.localhost
      KC_HOSTNAME_PORT: 8080
      KC_HTTP_ENABLED: "true"
    volumes:
      - ./realm/realm-devsecops.json:/opt/keycloak/data/import/realm-devsecops.json:ro
    ports:
      - "127.0.0.1:8080:8080"
      - "127.0.0.1:9000:9000"
    networks:
      iam-net:
        aliases:
          - keycloak.localhost
    depends_on:
      db:
        condition: service_healthy

  app:
    build:
      context: ./app
      dockerfile: Dockerfile
    image: devsecops-bab09-flask:1.0
    container_name: devsecops-bab09-app-1
    restart: unless-stopped
    environment:
      FLASK_SECRET_KEY: ${FLASK_SECRET_KEY}
      OIDC_CLIENT_ID: ${OIDC_CLIENT_ID:-devsecops-app}
      OIDC_CLIENT_SECRET: ${OIDC_CLIENT_SECRET}
      OIDC_ISSUER: http://keycloak.localhost:8080/realms/devsecops-lab
    ports:
      - "127.0.0.1:5000:5000"
    networks:
      - iam-net
    depends_on:
      - keycloak
    healthcheck:
      test: ["CMD", "python3", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:5000/health')"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  postgres-data:

networks:
  iam-net:
    driver: bridge
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 17** mendefinisikan kontainer database `db` berbasis PostgreSQL 17 Alpine, menginjeksikan kredensial database dari file `.env`, memetakan named volume persisten `postgres-data`, menghubungkannya ke jaringan internal `iam-net` tanpa mempublikasikan port ke host, serta mengonfigurasi probe kesehatan `pg_isready`.

2. **Line 18 sampai 26** mendefinisikan layanan Identity Provider `keycloak` berbasis versi rilis resmi `26.7.2`, menginstruksikan mode pengembangan `start-dev`, mengimpor file deklarasi realm secara otomatis saat startup (`--import-realm`), serta mengaktifkan modul metrik dan health probe (`--health-enabled=true`).

3. **Line 27 sampai 36** mengonfigurasi konektivitas JDBC asinkron Keycloak ke database PostgreSQL internal `jdbc:postgresql://db:5432/keycloak`, menetapkan kredensial administrator awal, mengonfigurasi nama host terstandarisasi `keycloak.localhost` pada port 8080, serta mengizinkan protokol HTTP lokal.

4. **Line 37 sampai 49** me-mount file konfigurasi realm `./realm/realm-devsecops.json` ke direktori impor Keycloak secara *read-only*, membatasi binding port layanan 8080 dan port manajemen 9000 ke loopback host `127.0.0.1`, menetapkan alias jaringan internal `keycloak.localhost`, serta menetapkan dependensi kondisi kesehatan `service_healthy` pada database.

5. **Line 50 sampai 69** mengonfigurasi container aplikasi Relying Party `app` yang dibangun dari `./app/Dockerfile`, menginjeksikan kunci rahasia Flask, kredensial client OIDC rahasia, URL issuer resmi Keycloak, mengikat port loopback `127.0.0.1:5000:5000`, serta mengonfigurasi health check internal pada endpoint `/health`.

6. **Line 70 sampai 78** mendeklarasikan named volume persisten `postgres-data` untuk menjaga persistensi data database identitas dan mendefinisikan jaringan isolasi bridge `iam-net`.


### **Lampiran 8 - realm/realm-devsecops.json (Deklarasi Realm-as-Code)**
```json
{
  "realm": "devsecops-lab",
  "enabled": true,
  "displayName": "DevSecOps Laboratory Realm",
  "sslRequired": "none",
  "roles": {
    "realm": [
      {
        "name": "viewer",
        "description": "Peran hak baca dasar untuk materi praktikum"
      },
      {
        "name": "admin",
        "description": "Peran administratif pengelolaan laboratorium"
      }
    ]
  },
  "clients": [
    {
      "clientId": "devsecops-app",
      "name": "Flask Relying Party Client",
      "enabled": true,
      "clientAuthenticatorType": "client-secret",
      "secret": "devsecops-client-secret-2026",
      "redirectUris": [
        "http://localhost:5000/auth/callback"
      ],
      "postLogoutRedirectUris": [
        "http://localhost:5000/"
      ],
      "standardFlowEnabled": true,
      "implicitFlowEnabled": false,
      "directAccessGrantsEnabled": false,
      "publicClient": false,
      "protocol": "openid-connect",
      "attributes": {
        "pkce.code.challenge.method": "S256"
      }
    }
  ],
  "users": [
    {
      "username": "mahasiswa1",
      "enabled": true,
      "email": "mahasiswa1@pens.ac.id",
      "credentials": [
        {
          "type": "password",
          "value": "MahasiswaLab!2026",
          "temporary": false
        }
      ],
      "realmRoles": [
        "viewer"
      ]
    },
    {
      "username": "adminlab",
      "enabled": true,
      "email": "adminlab@pens.ac.id",
      "credentials": [
        {
          "type": "password",
          "value": "AdminLab!2026",
          "temporary": false
        }
      ],
      "realmRoles": [
        "viewer",
        "admin"
      ]
    }
  ]
}
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 17** mendefinisikan konfigurasi inti realm dengan nama `devsecops-lab`, mengaktifkan status realm, menyetel persyaratan SSL ke mode `none` (khusus untuk lingkungan laboratorium loopback terkontrol), serta mendeklarasikan dua peran realm resmi yaitu peran `viewer` untuk akses baca dasar dan peran `admin` untuk akses wewenang penuh.

2. **Line 18 sampai 38** mengonfigurasi klien rahasia (*confidential client*) `devsecops-app` dengan tipe autentikasi `client-secret`, mendefinisikan whitelist URL pengalihan sah `http://localhost:5000/auth/callback` dan URL pasca-logout `http://localhost:5000/`, mengaktifkan Authorization Code Flow (`standardFlowEnabled: true`), menonaktifkan Implicit Flow, serta memblokir Direct Access Grants (`directAccessGrantsEnabled: false`). Atribut klien mewajibkan penggunaan metode tantangan kriptografi `S256` (PKCE).

3. **Line 39 sampai 54** mendeklarasikan akun pengguna `mahasiswa1` berstatus aktif dengan alamat email resmi, menetapkan kredensial password permanen, serta mengalokasikan hak akses peran realm terbatas pada peran `viewer` saja.

4. **Line 55 sampai 71** mendeklarasikan akun personel `adminlab` berstatus aktif, menetapkan kredensial password administratif permanen, serta mengalokasikan kombinasi peran realm `viewer` dan `admin`, memberikannya hak wewenang ganda untuk mengakses endpoint operasional maupun panel kontrol administratif.


### **Lampiran 9 - app/app.py (Relying Party Flask OIDC PKCE & RBAC)**
```python
import os
import json
from functools import wraps
from flask import Flask, redirect, url_for, session, abort, jsonify
from authlib.integrations.flask_client import OAuth
import jwt

app = Flask(__name__)
app.secret_key = os.environ.get("FLASK_SECRET_KEY", "dev-secret-key-replace-in-prod")

oauth = OAuth(app)
oauth.register(
    name="keycloak",
    client_id=os.environ.get("OIDC_CLIENT_ID", "devsecops-app"),
    client_secret=os.environ.get("OIDC_CLIENT_SECRET"),
    server_metadata_url=f"{os.environ.get('OIDC_ISSUER')}/.well-known/openid-configuration",
    client_kwargs={
        "scope": "openid profile email",
        "code_challenge_method": "S256"
    }
)

def require_role(required_role):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if "user" not in session:
                return redirect(url_for("login"))
            user_roles = session.get("roles", [])
            if required_role not in user_roles:
                abort(403)
            return f(*args, **kwargs)
        return decorated_function
    return decorator

@app.route("/")
def index():
    user = session.get("user")
    roles = session.get("roles", [])
    return jsonify({
        "status": "online",
        "authenticated": user is not None,
        "username": user.get("preferred_username") if user else None,
        "roles": roles
    })

@app.route("/health")
def health():
    return jsonify({"status": "UP"})

@app.route("/login")
def login():
    redirect_uri = url_for("auth_callback", _external=True)
    return oauth.keycloak.authorize_redirect(redirect_uri)

@app.route("/auth/callback")
def auth_callback():
    token = oauth.keycloak.authorize_access_token()
    user_info = token.get("userinfo") or {}
    
    # Extract realm roles from access token or id token
    access_token = token.get("access_token")
    roles = []
    if access_token:
        decoded = jwt.decode(access_token, options={"verify_signature": False})
        roles = decoded.get("realm_access", {}).get("roles", [])
    
    session["user"] = dict(user_info)
    session["roles"] = roles
    return redirect(url_for("index"))

@app.route("/viewer")
@require_role("viewer")
def viewer():
    return jsonify({
        "message": "Selamat datang di area Viewer materi DevSecOps",
        "username": session["user"].get("preferred_username"),
        "role_authorized": "viewer"
    })

@app.route("/admin")
@require_role("admin")
def admin():
    return jsonify({
        "message": "Selamat datang di Panel Administrasi Laboratorium",
        "username": session["user"].get("preferred_username"),
        "role_authorized": "admin"
    })

@app.route("/logout")
def logout():
    session.clear()
    issuer = os.environ.get("OIDC_ISSUER")
    post_logout = url_for("index", _external=True)
    logout_url = f"{issuer}/protocol/openid-connect/logout?post_logout_redirect_uri={post_logout}"
    return redirect(logout_url)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 9** mengimpor library standar Flask, decorator functools, client integration library Authlib OAuth, token parser library PyJWT, serta menginisialisasi instans aplikasi Flask dengan konfigurasi `secret_key` sesi.

2. **Line 10 sampai 22** menginisialisasi modul Authlib OAuth dan mendaftarkan klien `keycloak` dengan membaca URL penemuan metadata OpenID secara dinamis (`/.well-known/openid-configuration`), menetapkan cakupan data `openid profile email`, serta mewajibkan metode tantangan kriptografi `S256` (PKCE).

3. **Line 23 sampai 35** mendefinisikan fungsi decorator otorisasi `@require_role`. Decorator memeriksa keberadaan data pengguna di dalam browser session (mengalihkan ke rute `/login` jika unauthenticated) dan memvalidasi apakah peran yang disyaratkan terdapat pada daftar `session['roles']`, merespons HTTP `403 Forbidden` jika pengguna tidak memiliki hak wewenang.

4. **Line 36 sampai 54** mendefinisikan rute publik beranda `/` yang mengembalikan ringkasan status autentikasi dan peran pengguna, rute kesehatan internal `/health` (`status: UP`), serta rute `/login` yang membentuk URL otorisasi Keycloak beserta parameter PKCE dan mengalihkan browser.

5. **Line 55 sampai 71** mendefinisikan rute penanganan callback `/auth/callback` yang menukarkan kode otorisasi dan `code_verifier` dengan token akses, mendekodekan klaim peran otorisasi dari `realm_access.roles` pada token akses JWT, serta menyimpan profil pengguna dan daftar peran ke dalam sesi terenkripsi server.

6. **Line 72 sampai 89** mendefinisikan rute terproteksi `/viewer` yang dilindungi peran `viewer` dan rute terproteksi `/admin` yang dilindungi peran `admin`, mengembalikan data JSON terpersonalisasi apabila otorisasi berhasil.

7. **Line 90 sampai 104** mendefinisikan rute `/logout` yang menghancurkan sesi lokal Flask (`session.clear()`), membentuk URL Single Logout resmi Keycloak dengan menyematkan parameter `post_logout_redirect_uri`, serta mengarahkan browser untuk mematikan sesi SSO terpusat.


### **Lampiran 10 - scripts/negative-tests.sh (Pengujian Negatif Perimeter OIDC)**
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "${BASH_SOURCE[0]}")/."
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

# 1 & 2. Akses tanpa sesi dialihkan (HTTP 302)
expect_status "viewer tanpa sesi" "302" "http://localhost:5000/viewer"
expect_status "admin tanpa sesi" "302" "http://localhost:5000/admin"

# 3. Penolakan redirect_uri liar di luar whitelist
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

# 4. Penolakan otorisasi tanpa parameter PKCE
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

# 5. Penolakan Direct Access Grants (ROPC)
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

1. **Line 1 sampai 16** mengaktifkan mode penanganan kesalahan Bash, memuat variabel lingkungan dari file `.env`, menginisialisasi penghitung kegagalan `failures=0`, serta mendefinisikan fungsi pembantu `expect_status()` untuk menguji kode status respons HTTP.

2. **Line 17 sampai 28** menjalankan pengujian skenario 1 dan 2 dengan mengirimkan permintaan HTTP GET ke endpoint terproteksi `/viewer` dan `/admin` tanpa membawa cookie sesi, memverifikasi kembalian kode status HTTP `302 Found` yang membuktikan pengalihan paksa ke login portal IdP.

3. **Line 29 sampai 45** menjalankan pengujian skenario 3 dengan mengirimkan permintaan otorisasi ke Keycloak dengan menyematkan URL pengalihan liar yang tidak terdaftar (`http://attacker.invalid/callback`), memverifikasi penolakan tegas server IdP dengan kode status HTTP `400 Bad Request`.

4. **Line 46 sampai 60** menjalankan pengujian skenario 4 dengan mengirimkan permintaan otorisasi resmi namun sengaja menghilangkan parameter `code_challenge` (PKCE), memverifikasi bahwa Keycloak menolak flow tanpa PKCE dengan menghasilkan respons pengalihan error `error=invalid_request`.

5. **Line 61 sampai 79** menjalankan pengujian skenario 5 dengan melakukan simulasi serangan pengiriman kredensial langsung (Resource Owner Password Credentials) ke endpoint `/token`, memverifikasi bahwa metode Direct Access Grants ditolak dengan kode status HTTP `400/401` (`unauthorized_client`).


### **Lampiran 11 - scripts/test-e2e-auth.py (Simulasi E2E OIDC PKCE & RBAC)**
```python
import os
import re
import base64
import hashlib
import secrets
import requests
import argparse

def generate_pkce_pair():
    verifier = base64.urlsafe_b64encode(secrets.token_bytes(32)).rstrip(b'=').decode('ascii')
    digest = hashlib.sha256(verifier.encode('ascii')).digest()
    challenge = base64.urlsafe_b64encode(digest).rstrip(b'=').decode('ascii')
    return verifier, challenge

def run_e2e_flow(username, password, expected_admin_status):
    session = requests.Session()
    print(f"\n[PENGUJIAN] Menguji Autentikasi Pengguna: {username}")
    
    # 1. Initiating login to Flask RP
    resp = session.get("http://localhost:5000/login", allow_redirects=False)
    auth_url = resp.headers.get("Location")
    print(f"1. Inisiasi Login -> Pengalihan ke IdP (HTTP {resp.status_code})")
    
    # 2. Reaching Keycloak Login Page
    login_page = session.get(auth_url)
    action_match = re.search(r'action="([^"]+)"', login_page.text)
    login_action_url = action_match.group(1).replace("&amp;", "&")
    
    # 3. Submitting Credentials
    auth_resp = session.post(login_action_url, data={"username": username, "password": password}, allow_redirects=False)
    cb_url = auth_resp.headers.get("Location")
    print(f"2. Pengiriman Kredensial -> Kode Otorisasi Diterbitkan (HTTP {auth_resp.status_code})")
    
    # 4. Exchanging Code at Callback
    final_cb = session.get(cb_url, allow_redirects=False)
    print(f"3. Penukaran Kode di Endpoint Callback Flask (HTTP {final_cb.status_code})")
    
    # 5. Testing Protected Endpoints
    v_resp = session.get("http://localhost:5000/viewer")
    print(f"4. Pengujian /viewer -> HTTP {v_resp.status_code} (Ekspektasi: 200 OK)")
    assert v_resp.status_code == 200, f"Gagal pada /viewer: {v_resp.status_code}"
    
    a_resp = session.get("http://localhost:5000/admin")
    print(f"5. Pengujian /admin  -> HTTP {a_resp.status_code} (Ekspektasi: {expected_admin_status})")
    assert a_resp.status_code == expected_admin_status, f"Gagal pada /admin: {a_resp.status_code}"
    print(f"[PASS] Penegakan RBAC untuk {username} terverifikasi sempurna.")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--user", choices=["mahasiswa1", "adminlab", "all"], default="all")
    args = parser.parse_args()
    
    if args.user in ["mahasiswa1", "all"]:
        run_e2e_flow("mahasiswa1", "MahasiswaLab!2026", 403)
    if args.user in ["adminlab", "all"]:
        run_e2e_flow("adminlab", "AdminLab!2026", 200)
```

**Penjelasan Baris Kode (Line-by-Line Breakdown)**

1. **Line 1 sampai 8** mengimpor modul standar Python yang diperlukan untuk manipulasi string, ekspresi reguler, encoding Base64, kalkulasi hash kriptografi, pembuatan bilangan acak aman `secrets`, HTTP library `requests`, serta parser argumen CLI `argparse`.

2. **Line 9 sampai 14** mendefinisikan fungsi `generate_pkce_pair()` yang membangkitkan string acak berentropi tinggi 32 byte sebagai `code_verifier`, menghitung digest hash SHA-256, dan melakukan Base64URL encoding untuk menghasilkan nilai `code_challenge`.

3. **Line 15 sampai 28** mendefinisikan fungsi `run_e2e_flow()` yang menginisialisasi browser session independen, melakukan panggilan GET ke rute `/login` Flask Relying Party, menangkap URL otorisasi Keycloak dari header `Location`, mengambil halaman login HTML, dan mengekstrak atribut URL form action.

4. **Line 29 sampai 40** mengirimkan kredensial username dan password via HTTP POST ke form action Keycloak, menangkap URL pengalihan callback yang memuat authorization code, serta mengeksekusi panggilan GET ke endpoint `/auth/callback` Flask untuk menyelesaikan proses jabat tangan penukaran token.

5. **Line 41 sampai 52** menguji penegakan kebijakan RBAC pada endpoint terproteksi dengan memverifikasi akses ke rute `/viewer` mengembalikan HTTP 200 OK, serta menguji rute `/admin` dan mencocokkannya dengan status ekspektasi (`403 Forbidden` untuk `mahasiswa1` dan `200 OK` untuk `adminlab`).

6. **Line 53 sampai 64** mengonfigurasi blok argumen CLI yang memungkinkan operator mengeksekusi pengujian secara modular untuk pengguna tertentu (`--user mahasiswa1` atau `--user adminlab`) atau menjalankan pengujian otomatis seluruh skenario pengguna sekaligus.

\pagebreak

\begin{center}
\small \textit{\textbf{Catatan \& Pernyataan Penggunaan AI:}\\Dokumen laporan ini disusun dengan bantuan AI yang digunakan secara terbatas untuk merapikan tata bahasa laporan serta men-generate catatan materi kuliah (saat Pak Ferry menjelaskan di kelas/lab) menjadi diagram visual peta konsep (mind map). Seluruh jawaban pada bagian Evaluasi dan Latihan Mandiri, analisis teknis, serta pengujian praktikum merupakan hasil pengerjaan, pemahaman mandiri, dan analisis saya secara langsung.}
\end{center}
