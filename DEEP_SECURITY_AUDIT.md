# Audit Keamanan Kode & Arsitektur Lanjutan (Deep Security Audit) Wibuku 1.4.4

Dokumen ini merupakan **Laporan Audit Keamanan Lanjutan (*Deep Security Audit Report*)** yang menganalisis aspek keamanan tingkat kode, penyimpanan data lokal, interaksi komponen Android, serta manajemen sesi pada aplikasi **Wibuku v1.4.4** (`Wibuku_1.4.4.xapk`).

---

## 1. Audit Penyimpanan Data Lokal & SharedPreferences (`wibuku.app.wibuku.model.pref`)

### 1.1 Temuan: Penyimpanan Unencrypted SharedPreferences (`ParentPref.smali`)
* **Lokasi Kode**: `wibuku/app/wibuku/model/pref/ParentPref.smali` & `mr3.smali`
* **Analisis**:
  Aplikasi menggunakan kelas pembungkus `ParentPref` yang menginstansiasi `SharedPreferences` standar dengan mode `MODE_PRIVATE` (`0`):
  ```smali
  const/4 v1, 0x0
  invoke-virtual {v0, p1, v1}, Landroid/content/Context;->getSharedPreferences(Ljava/lang/String;I)Landroid/content/SharedPreferences;
  ```
* **Resiko Keamanan**:
  Meskipun `MODE_PRIVATE` mencegah aplikasi lain mengakses file secara langsung pada perangkat non-root, data di dalam SharedPreferences disimpan dalam bentuk plaintext XML (`/data/data/wibuku.app.wibuku/shared_prefs/`).
* **Data Sensitif Teridentifikasi**:
  - `pending_recharge_token`: Token sesi pembayaran/recharge.
  - `notif_chat_friend_mute_*`: Konfigurasi filter obrolan.
  - Data preferensi akun pengguna dan flag pengaturan aplikasi.
* **Rekomendasi Remediasi**:
  Migrasikan penyimpanan token dan preferensi sensitif ke `EncryptedSharedPreferences` (bagian dari Jetpack Security / `androidx.security.crypto`) yang memanfaatkan Android Keystore System untuk mengenkripsi file XML secara transparan.

---

## 2. Audit Komponen Android & Antarmuka WebView (`ClanHallFragment` & `PremiumNewFragment`)

### 2.1 Temuan: Eksposur Method Native Melalui JavaScript Interface Tanpa Restriction
* **Lokasi Kode**:
  - `ClanHallFragment.smali` -> Interface `WIBUKU_NATIVE` (`z60.smali`)
  - `PremiumNewFragment.smali` -> Interface `Android` (`is3.smali`)
* **Analisis Metode `z60.smali`**:
  ```java
  public final void applyWatchState(String p1) { ... }
  public final void clearWatchState() { ... }
  public final void closeHall() { ... }
  public final long getWatchPositionMs() { ... }
  public final void hallLiveChatLine(String p1, String p2, boolean p3) { ... }
  public final void openMemberProfile(long p1) { ... }
  public final void pickWatchAnime() { ... }
  ```
* **Vulnerability Flow**:
  1. WebView mengaktifkan `setJavaScriptEnabled(true)`.
  2. Native Bridge terdaftar via `addJavascriptInterface(new z60(this), "WIBUKU_NATIVE")`.
  3. WebView tidak membatasi skema URL / Origin Host di `shouldOverrideUrlLoading`.
* **Skenario Eksploitasi**:
  Jika user membuka link eksternal atau WebView mengarahkan ke halaman dengan konten yang disisipi XSS, skrip JavaScript jahat dapat langsung mengeksekusi `window.WIBUKU_NATIVE.openMemberProfile(userId)` atau `window.WIBUKU_NATIVE.hallLiveChatLine(...)` untuk mengirimkan obrolan palsu atas nama user secara otomatis.
* **Rekomendasi Remediasi**:
  1. Batasi domain yang boleh dimuat oleh WebView.
  2. Implementasikan pengecekan `Uri.parse(url).getHost().endsWith("wibuku.app")` sebelum mengaktifkan atau mengeksekusi fungsi JavaScript Interface.

---

## 3. Audit Pengolahan Basis Data Lokal SQLite (`RoomDatabase` & `SQLiteOpenHelper`)

### 3.1 Temuan: Query SQLite Dinamis & Parameterization Review
* **Lokasi Kode**: `d10.smali` (Tabel `chat_friend`), `e10.smali` (Tabel `notifications`)
* **Analisis**:
  Pengujian terhadap kueri lokal menunjukkan aplikasi menggunakan `RoomDatabase` / SQLite terparameterisasi secara konsisten (`?` placeholder):
  ```sql
  SELECT * FROM chat_friend WHERE conv_id = ? ORDER BY id ASC
  DELETE FROM notifications WHERE id = ?
  SELECT COUNT(*) FROM notifications WHERE user_id = ? AND is_read = 0
  ```
* **Hasil Assessment**:
  **SAFE (Aman dari SQL Injection)**. Tidak ditemukan penggunaan penggabungan string (*string concatenation*) berbahaya pada kueri SQLite internal.

---

## 4. Audit Manajemen Sesi & Replay Protection (`cpalette` & `X-Panel-Auth`)

### 4.1 Temuan: Penggunaan Timestamp Client-Side pada Authentication (`cpalette`)
* **Lokasi Kode**: `vj1.java`, `wj1.java`, `gc4.java`
* **Analisis**:
  Saat melakukan request `user/login`, aplikasi menghitung token `cpalette` menggunakan `System.nanoTime()` dan salt `gc4.s`:
  ```java
  cpalette = SHA256(nanoTime + gc4.s);
  ```
* **Kelemahan Arsitektur**:
  1. `System.nanoTime()` mengukur waktu sejak boot perangkat dan bervariasi antar perangkat.
  2. Karena salt `gc4.s` bersifat statis dan tersimpan di dalam aplikasi (`40248069404125886141037650881278m`), penyerang yang mengetahui salt ini dapat dengan mudah memproduksi nilai `cpalette` yang valid dari skrip eksternal (Python/Curl) untuk melakukan otomatisasi request login.
* **Rekomendasi Remediasi**:
  Gunakan mekanisme `Nonce` berbasis server atau challenge-response token (HMAC-SHA256 dengan time-window berbasis server / TOTP) daripada bergantung pada salt statis di sisi client.

---

## 5. Ringkasan Audit & Matriks Risiko Lanjutan

| Area Audit | Status Keamanan | Risk Level | Catatan Ringkas |
| :--- | :--- | :--- | :--- |
| **SharedPreferences** | Unencrypted Plaintext XML | **MEDIUM** | Token recharge tersimpan di `/data/data/wibuku.app.wibuku/shared_prefs/`. |
| **WebView Native Bridge** | Missing Origin Validation | **HIGH** | `WIBUKU_NATIVE` dapat dipanggil dari mana saja jika WebView memuat URL eksternal. |
| **SQLite Local Database** | Parameterized Queries | **LOW (SAFE)** | Penggunaan kueri terparameterisasi (`?`) mencegah SQL Injection lokal. |
| **Replay Protection (`cpalette`)**| Client-Calculated Salt | **MEDIUM** | Salt statis tersimpan di client sehingga kalkulasi `cpalette` dapat ditiru. |
| **Network Cleartext Traffic** | Enabled in Manifest | **MEDIUM** | `android:usesCleartextTraffic="true"` mengizinkan koneksi unencrypted. |

---

## 6. Kesimpulan

Laporan ini mengonfirmasi bahwa kendati basis data lokal (SQLite) telah menerapkan praktik kueri aman, terdapat **risiko arsitektural** pada penggunaan WebView JavaScript Interface tanpa pembatasan domain (*Origin Restriction*), penyimpanan SharedPreferences tanpa enkripsi (`EncryptedSharedPreferences`), serta dependensi pada salt statis client-side untuk verifikasi sesi. Disarankan agar pengembang segera menerapkan perbaikan sesuai rekomendasi yang tercantum.
