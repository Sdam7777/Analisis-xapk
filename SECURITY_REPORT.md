# Laporan Kunci Dekripsi & Audit Celah Keamanan (Vulnerabilities) Wibuku 1.4.4

Dokumen ini berisi spesifikasi teknis mengenai **Kunci Dekripsi Aplikasi** serta hasil audit **Celah Keamanan (*Security Vulnerabilities*)** mendalam pada aplikasi Android **Wibuku v1.4.4** (`Wibuku_1.4.4.xapk`).

---

## 1. Kunci Dekripsi Aplikasi & Formula Reversal

Aplikasi Wibuku menggunakan enkripsi string kustom berbasis hash SHA-256 dan substitusi karakter 5-pass (`kf3` & `ce5`). Kunci-kunci induk yang diekstrak adalah sebagai berikut:

### 1.1 Master Seed & Secret Keys

1. **Master Seed Key (`ce5.h("NDQyMzEy")`)**:
   - **Encoded String**: `NDQyMzEy` (Base64)
   - **Master Seed Plaintext**: `442312`
   - **SHA-256 Digest**: `0e0294e9f565fa92629b398df9033f789e9f60f64b85c2c77f0a9df4953c306e`

2. **Backend API Base URL (`gc4.g`)**:
   - **Obfuscated Payload**: `34268224513506522443312558274442078368493327562450420589665040028525229708snunsyfvjyvo:/.rjht`
   - **Decrypted Result**: `https://api.wibuku.app/`

3. **CDN Media Server URL (`gc4.a`)**:
   - **Obfuscated Payload**: `41259931542`
   - **Decrypted Result**: `https://cdn.wibuku.app/`

4. **Web & Payment Gateway Base URL (`gc4.k`)**:
   - **Obfuscated Payload**: `33268224554302582441412781685444068064444923562457037650v.wt:mjoakz/ecap`
   - **Decrypted Result**: `https://wibuku.app/`

5. **Internal Salt Secrets**:
   - `gc4.r`: Signature Salt Internal (`49258261440376503693`)
   - `gc4.s`: Replay Protection Secret (`40248069404125886141037650881278m`)
   - `gc4.t`: Login Hash Salt (`4020856949432584614749444229i8ghh8M8g`)

---

## 2. Audit Celah Keamanan Mendalam (Advanced Vulnerabilities)

---

### Celah 1: Insecure WebView Javascript Interface Origin Validation (High Severity)
* **Vulnerability ID**: `WBK-VULN-001`
* **Kategori**: Insecure WebView Implementation / Cross-Site Scripting Bridge
* **Tingkat Keparahan**: **HIGH** (Tinggi)
* **Deskripsi**:
  Aplikasi mengekspos Javascript Interface `WIBUKU_NATIVE` (`z60.smali` / `ClanHallFragment`) dan `Android` (`is3.smali` / `PremiumNewFragment`) ke WebView tanpa memvalidasi domain URL yang dimuat (*URL Origin Check*).
* **Komponen Terdampak**:
  - `ClanHallFragment` (`WIBUKU_NATIVE` -> `openMemberProfile`, `hallLiveChatLine`, `applyWatchState`, `pickWatchAnime`)
  - `PremiumNewFragment` (`Android` -> `saveQrImage`)
* **Dampak**:
  Jika WebView memuat halaman web eksternal atau terkena serangan *Man-in-the-Middle* (MITM), halaman penyerang dapat mengeksekusi metode native Android untuk memanipulasi obrolan clan, membuka profil pengguna lain, atau mengeksekusi aksi berbasis UI secara otomatis.
* **Mitigasi**:
  Lakukan verifikasi domain terpercaya (misal hanya domain `*.wibuku.app`) pada callback `shouldOverrideUrlLoading` sebelum mendaftarkan `addJavascriptInterface`.

---

### Celah 2: Hardcoded Master Decryption Seed pada Client (High Severity)
* **Vulnerability ID**: `WBK-VULN-002`
* **Kategori**: Cryptographic Issues / Hardcoded Secrets
* **Tingkat Keparahan**: **HIGH** (Tinggi)
* **Deskripsi**:
  Master seed (`442312`) dan logika dekripsi `kf3` tertanam penuh di dalam DEX (*client-side*). Semua string sensitif (API Key, Salt, Base URL) disandi dengan kunci statis yang sama.
* **Dampak**:
  Penyerang dapat dengan mudah mengekstrak seluruh URL rahasia, kunci salt internal, dan endpoint tersembunyi tanpa memerlukan akses root atau intersept jaringan HTTPS.
* **Mitigasi**:
  Gunakan Android Keystore System / Native Library (`.so`) dengan NDK obfuscation, atau pindahkan sensitivitas kunci ke proses autentikasi server (Token Exchange).

---

### Celah 3: Keberadaan Endpoint Dev Login & Bypass Header (High Severity)
* **Vulnerability ID**: `WBK-VULN-003`
* **Kategori**: Broken Authentication / Exposed Debug Features
* **Tingkat Keparahan**: **HIGH** (Tinggi)
* **Deskripsi**:
  Aplikasi menyertakan endpoint `POST user/devlogin` pada build rilis publik:
  ```java
  @nk3("user/devlogin")
  @wn1
  @y52({"X-Skip-Panel-Auth: 1"})
  Object w(@vg1("username") String str, @vg1("password") String str2, ...);
  ```
  Endpoint ini menggunakan header `@Headers({"X-Skip-Panel-Auth: 1"})` yang melewati pengecekan token JWT/Session normal.
* **Dampak**:
  Jika server production tidak menolak request ke `/user/devlogin`, penyerang dapat melakukan brute-force atau meng-access akun administrator/developer tanpa autentikasi user normal.
* **Mitigasi**:
  Hapus endpoint `user/devlogin` dan flag `X-Skip-Panel-Auth` dari build production/release XAPK.

---

### Celah 4: Izin Traffic HTTP Plaintext (`android:usesCleartextTraffic="true"`) (Medium Severity)
* **Vulnerability ID**: `WBK-VULN-004`
* **Kategori**: Network Security Configuration / Plaintext Communication
* **Tingkat Keparahan**: **MEDIUM** (Sedang)
* **Deskripsi**:
  `AndroidManifest.xml` secara eksplisit mengaktifkan `android:usesCleartextTraffic="true"`.
* **Dampak**:
  Aplikasi diizinkan untuk mengirimkan data HTTP tanpa enkripsi TLS. Dalam jaringan Wi-Fi publik, penyerang dapat mengintip (*sniffing*) atau memodifikasi paket data yang dikirim melalui HTTP plaintext.
* **Mitigasi**:
  Atur `android:usesCleartextTraffic="false"` pada `AndroidManifest.xml` dan terapkan `networkSecurityConfig` yang mewajibkan seluruh koneksi menggunakan HTTPS.

---

### Celah 5: Client-Side Parameter Control (`admin_override` Parameter) (Medium Severity)
* **Vulnerability ID**: `WBK-VULN-005`
* **Kategori**: Broken Access Control / Insecure Direct Object Reference
* **Tingkat Keparahan**: **MEDIUM** (Sedang)
* **Deskripsi**:
  Pada endpoint penolakan pertemanan `POST friend/reject`, client mengirimkan parameter boolean `admin_override`:
  ```java
  @nk3("friend/reject")
  @wn1
  Object e0(@vg1("request_id") long j, @vg1("admin_override") boolean z, @q52("device_hash") String str, ...);
  ```
* **Dampak**:
  Pengguna biasa dapat memodifikasi payload HTTP dan mengirimkan `admin_override=true` untuk memotong batasan hak akses pertemanan atau membatalkan relasi user lain jika verifikasi role di sisi server tidak ketat.
* **Mitigasi**:
  Hak akses admin harus ditentukan sepenuhnya di sisi server (*Server-Side Authorization*) berdasarkan Token Session (`X-Panel-Auth`), bukan dari parameter request client.

---

### Celah 6: Terbuka & Dapat Di-spoofingnya Identifikasi Perangkat (`device_hash`) (Medium Severity)
* **Vulnerability ID**: `WBK-VULN-006`
* **Kategori**: Insecure Device Fingerprinting / Spoofing
* **Tingkat Keparahan**: **MEDIUM** (Sedang)
* **Deskripsi**:
  `device_hash` dibuat secara deterministik dari `ANDROID_ID` perangkat. Tidak ada verifikasi Hardware Attestation (Google Play Integrity / SafetyNet) yang digunakan untuk memvalidasi keaslian hash ini.
* **Dampak**:
  Penyerang dapat dengan mudah melakukan spoofing `device_hash` secara acak untuk membuat ribuan akun palsu, melakukan spamming di chat global/clan, atau menghindari pembatasan rate limit berbasis perangkat.
* **Mitigasi**:
  Integrasikan Play Integrity API untuk memastikan request benar-benar berasal dari aplikasi resmi yang berjalan pada perangkat Android valid yang tidak dimodifikasi.

---

### Celah 7: Kurangnya SSL Certificate Pinning (Low to Medium Severity)
* **Vulnerability ID**: `WBK-VULN-007`
* **Kategori**: Transport Layer Security / Missing Certificate Pinning
* **Tingkat Keparahan**: **MEDIUM** (Sedang)
* **Deskripsi**:
  Aplikasi tidak menerapkan `CertificatePinner` pada OkHttpClient (`aj3.java` / `s4.java`). Konfigurasi Network Security hanya mengandalkan CA default sistem Android.
* **Dampak**:
  Lalu lintas HTTPS antara aplikasi Wibuku dan server `api.wibuku.app` dapat di-intersept (*Man-in-the-Middle / MITM*) menggunakan CA Certificate khusus pada perangkat ter-root atau emulator (menggunakan tools seperti Burp Suite / HTTP Toolkit).
* **Mitigasi**:
  Terapkan SSL/TLS Certificate Pinning pada `OkHttpClient` untuk domain `api.wibuku.app` dan `cdn.wibuku.app`.

---

## 3. Matriks Ringkasan Celah Keamanan

| Vuln ID | Nama Celah Keamanan | Keparahan | Komponen Terdampak | Status Remediation |
| :--- | :--- | :--- | :--- | :--- |
| **WBK-VULN-001** | Insecure WebView Javascript Interface | **HIGH** | `WIBUKU_NATIVE`, `ClanHallFragment` | Perlu Perbaikan |
| **WBK-VULN-002** | Hardcoded Master Decryption Seed | **HIGH** | `kf3.java`, `ce5.java`, `gc4.java` | Perlu Perbaikan |
| **WBK-VULN-003** | Exposed Dev Login & Bypass Header | **HIGH** | `user/devlogin`, `bl3.java` | Perlu Perbaikan |
| **WBK-VULN-004** | Allowed Cleartext HTTP Traffic | **MEDIUM** | `AndroidManifest.xml` | Perlu Perbaikan |
| **WBK-VULN-005** | Client-Controlled `admin_override` | **MEDIUM** | `friend/reject`, `bl3.java` | Perlu Perbaikan |
| **WBK-VULN-006** | Spoofable Device Fingerprint Hash | **MEDIUM** | `device_hash`, `lr3.java` | Perlu Perbaikan |
| **WBK-VULN-007** | Missing SSL Certificate Pinning | **MEDIUM** | Network Stack (`aj3.java`) | Perlu Perbaikan |

---

## 4. Kesimpulan & Rekomendasi Tambahan

Sistem keamanan aplikasi **Wibuku 1.4.4** memiliki beberapa celah kritis (*High Severity*) pada sisi WebView Interface yang tidak divalidasi, pengalokasian kunci dekripsi statis di sisi client, serta terbukanya endpoint developer login (`user/devlogin`). Diperlukan perbaikan komprehensif pada arsitektur autentikasi, pembatasan asal domain WebView (*Origin Validation*), serta pemindahan verifikasi hak akses seluruhnya ke sisi server (*Server-Side Authorization*).
