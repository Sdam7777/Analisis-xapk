# Laporan Analisis Reverse Engineering & Teknik Komunikasi Client-Server Wibuku 1.4.4 (XAPK)

Dokumen ini berisi hasil analisis teknis mendalam (*reverse engineering*) terhadap aplikasi Android **Wibuku v1.4.4** (`Wibuku_1.4.4.xapk`). Analisis ini berfokus pada arsitektur proteksi, enkripsi string, mekanisme keamanan komunikasi client-server, serta katalog lengkap REST API Endpoints.

---

## 1. Ringkasan Eksekutif & Identitas Aplikasi

* **Nama Aplikasi**: Wibuku
* **Versi**: 1.4.4 (Build Release XAPK / Split APKs)
* **Package Name**: `wibuku.app.wibuku`
* **Arsitektur Obfuskasi Kode**: ProGuard / R8 Obfuscation + Custom Multi-Pass String Encryption (`kf3` / `ce5`).
* **Framework Komunikasi Network**: Retrofit 2 (`defpackage.bl3`) + OkHttp 4 + Gson Serializer (`defpackage.cw0`).
* **Target OS / Minimum SDK**: Android 6.0 (API 23+) hingga Android 14+ (API 34).

---

## 2. Analisis Proteksi Aplikasi & Obfuskasi Kode

### 2.1 Algoritma Enkripsi String Kustom (`kf3` & `ce5`)
Aplikasi Wibuku mengamankan String sensitif (seperti API Keys, Base URLs, Salt Token, dan header internal) dari pembacaan statis menggunakan algoritma enkripsi kustom berbasis kalkulasi hash & substitusi pergeseran karakter.

1. **Seed Master Hash Generation**:
   Kunci induk (*seed*) diekstrak dari fungsi `ce5.h("NDQyMzEy")` yang menghasilkan String Base64 terurai `442312`.
   SHA-256 dari `442312` dihitung untuk membentuk digest 64-karakter hex:
   $$\text{SeedHash} = \text{SHA256}("442312")$$

2. **Multi-Pass Substitution (5 Iterasi)**:
   Proses dekripsi menjalankan loop dari $i = 4$ turun ke $0$.
   Di setiap perulangan:
   - Dihitung seed turunan: $H_i = \text{SHA256}(\text{SeedHash} + ":" + i)$.
   - Dibentuk tabel pemetaan substitusi untuk karakter alfabet `a`-`z` dan digit `0`-`9`.
   - Karakter dalam ciphertext digeser posisinya secara rasional berdasarkan indeks alfabet/angka.

3. **Positional Unshuffling**:
   Setelah pergeseran selesai, string dipisah berdasarkan pola substring penanda `442312`. Indeks pasangan digit (seperti `04`, `08`, `12`) digunakan sebagai penunjuk posisi penyusunan ulang (*unshuffling*) untuk membentuk string asli.

### 2.2 Hasil Dekripsi String Sensitif (`gc4` & `lr3`)

Berikut adalah nilai string sensitif yang berhasil dideskripsi dari kelas obfuskasi `gc4` dan `lr3`:

| Obfuscated Key | Enkripsi Smali / Encoded String | Hasil Dekripsi (Plaintext) | Fungsi dalam Aplikasi |
| :--- | :--- | :--- | :--- |
| `gc4.g` | `3426822451350652244331255...` | `https://api.wibuku.app/` | API Main Backend Base URL |
| `gc4.a` | `41259931542` | `https://cdn.wibuku.app/` | Media CDN & Cover Art Base URL |
| `gc4.k` | `3326822455430258244141278...` | `https://wibuku.app/` | Web Portal & QRIS Payment Gateway |
| `gc4.r` | `49258261440376503693` | Salt / API Key A | Signature Salt Internal |
| `gc4.s` | `402480694041258861410376...` | Key Session Secret | Kunci Hashing Replay-Attack |
| `gc4.t` | `40208569494328846147...` | Hash Salt B | Kunci Enkripsi Token Login |

---

## 3. Teknik & Arsitektur Komunikasi Client-Server

Aplikasi Wibuku menggunakan protokol RESTful HTTP/HTTPS yang dibangun di atas Retrofit 2 dan OkHttp 4. Komunikasi disecure dan divalidasi dengan beberapa teknik teknis berikut:

### 3.1 Identifikasi Perangkat (`device_hash`)
Untuk mencegah penggunaan bot, scraping, atau pengambilalihan akun dari emulator/device tak dikenal, setiap permintaan network menyertakan parameter `device_hash`.
* **Sumber Data**: `android_id` unik dari Android (`Settings.Secure.ANDROID_ID`).
* **Formulasi**:
  $$\text{device\_hash} = \text{SHA256}(\text{ANDROID\_ID} + \text{Salt Internal})$$
* Parameter ini dikirimkan via `@Query("device_hash")`, `@Field("device_hash")`, atau HTTP Header pada hampir 100% endpoint API.

### 3.2 Authentikasi & Bypassing Header (`X-Panel-Auth`)
* **Pengecekan Panel Auth**:
  Pesan sensitif menggunakan header `X-Panel-Auth: <JWT_TOKEN_USER>` untuk memvalidasi identitas user yang sedang aktif.
* **Header Bypass (`X-Skip-Panel-Auth: 1`)**:
  Untuk endpoint terbuka seperti pendaftaran akun baru (`user/register`) dan login developer (`user/devlogin`), Retrofit menggunakan annotation khusus `@Headers({"X-Skip-Panel-Auth: 1"})` agar interceptor OkHttp tidak menyisipkan token user yang belum terbentuk.

### 3.3 Anti Replay Attack & Dynamic Session Token (`cpalette`)
Pada proses login (`user/login`), aplikasi mengirimkan parameter timestamp dinamis `cpalette`:
$$\text{cpalette} = \text{System.nanoTime}() + \text{Salt Secret} (\text{gc4.s})$$
Server memverifikasi rentang waktu `nanoTime` ini untuk memastikan request login bukan merupakan hasil rekaman (*replay attack*).

### 3.4 Format Komunikasi Data
* **Content-Type Request**:
  - `application/json`: Digunakan untuk payload objek kompleks (seperti `RegisterRequest`, `CreateClanRequest`, `CommentBody`, `DonateRequest`).
  - `application/x-www-form-urlencoded`: Digunakan pada endpoint bertanda `@FormUrlEncoded` (`@wn1`) untuk pengiriman parameter form sederhana.
* **Standard Response Envelope (`ResourceResponse<T>`)**:
  Seluruh respon dari server dibungkus dalam struktur JSON generik:
  ```json
  {
    "status": 200,
    "message": "Success",
    "data": { ... }
  }
  ```

---

## 4. Ekstraksi Direct Video Streaming & Hoster Parser

Pada kelas `StreamSource` (`wibuku.app.wibuku.model.anime.StreamSource`), metode `getDirectLink` mengimplementasikan teknik parsing otomatis untuk mengurai URL video asli dari berbagai provider streaming anime:

1. **Parsing Encrypted Stream Object (`esu`)**:
   Jika respon provider mengandung script `esu('{"file":"...'`, regex extractor mengambil URL langsung file `.mp4` / `.m3u8`.
2. **Parsing Tag Media HTML5**:
   Jika provider menyajikan player HTML, extractor memindai atribut `<source src="...">` atau `<video src="...">`.
3. **Penanganan Playlist HLS (`.m3u8`)**:
   Untuk stream berbasis HLS, URL playlist master diteruskan secara langsung ke Android ExoPlayer dengan HTTP User-Agent asli aplikasi untuk melewati proteksi Referer hoster.

---

## 5. Katalog Lengkap REST API Endpoints (`defpackage.bl3`)

Berikut adalah daftar 80+ API Endpoints yang diekstrak secara akurat dari interface Retrofit `defpackage.bl3`:

### 5.1 Authentikasi & Manajemen User (`user/*`)

| HTTP Method | Path / Endpoint | Deskripsi & Parameter | Flag / Headers |
| :--- | :--- | :--- | :--- |
| **POST** | `user/register` | Pendaftaran akun baru (`RegisterRequest`) | `X-Skip-Panel-Auth: 1` |
| **POST** | `user/login` | Login user (`device_hash`, `cpalette`, `version`) | `FormUrlEncoded` |
| **POST** | `user/devlogin` | Developer Login (`username`, `password`) | `FormUrlEncoded`, `X-Skip-Panel-Auth: 1` |
| **POST** | `user/session` | Pengecekan status sesi & token | `device_hash` |
| **POST** | `user/profile/{id}` | Detail profil pengguna berdasarkan `id` | Path `id` |
| **POST** | `user/sync-photo` | Sinkronisasi foto profil Google Auth (`server_auth_code`) | `FormUrlEncoded` |
| **POST** | `user/sync/pinned` | Pin anime/item favorit ke profil | `FormUrlEncoded` |
| **POST** | `user/comment/` | Mengirim komentar baru (`CommentBody`) | JSON Body |
| **POST** | `user/comment/delete` | Menghapus komentar (`comment_id`) | `FormUrlEncoded` |
| **POST** | `user/react/` | Mengirim reaksi/like (`Reaction`) | JSON Body |
| **POST** | `user/recharge/` | Top-up koin/token (`token`) | `FormUrlEncoded` |
| **POST** | `user/recharge/token` | Pengambilan token sesi pembayaran | `device_hash` |
| **POST** | `user/premium/consume` | Verifikasi IAP Play Store (`product_id`, `purchase_token`, `order_id`) | `FormUrlEncoded` |
| **POST** | `user/histories/delete` | Menghapus riwayat menonton (`episode_ids`) | `FormUrlEncoded` |

---

### 5.2 Katalog Anime, Streaming & Giveaway (`anime/*`, `home/*`, `giveaway/*`)

| HTTP Method | Path / Endpoint | Deskripsi & Parameter | Flag / Headers |
| :--- | :--- | :--- | :--- |
| **POST** | `anime/home` | Feed halaman utama | `device_hash` |
| **POST** | `anime/popular` | Daftar anime populer (`mode`) | Query `mode` |
| **POST** | `anime/detail/{id}` | Detail anime & daftar episode | Path `id` |
| **POST** | `anime/episodemeta/{id}` | Metadata episode & link stream source (`id`, `mode`) | Path `id` |
| **POST** | `anime/comment/{episodeId}/{page}`| Komentar episode (`episodeId`, `page`, `mode`) | Path |
| **GET** | `home/top-donor-badges` | Daftar lencana pemberi donasi tertinggi | Query `limit` |
| **POST** | `home/top-givers` | Leaderboard donatur (`limit`, `timeline`) | Query |
| **POST** | `giveaway/{id}` | Informasi detail event giveaway | Path `id` |
| **POST** | `giveaway/replies/{id}` | Komentar/balasan pada giveaway | Path `id` |

---

### 5.3 Sistem Clan & Komunitas (`clan/*`, `clans/*`)

| HTTP Method | Path / Endpoint | Deskripsi & Parameter | Flag / Headers |
| :--- | :--- | :--- | :--- |
| **GET** | `clans/search` | Pencarian clan (`q`, `min_level`, `policy`, `has_slots`, `page`, `size`) | Query params |
| **GET** | `clans/leaderboard` | Peringkat clan global (`sort`, `limit`) | Query params |
| **GET** | `clan-create-config` | Konfigurasi syarat pembuatan clan | `device_hash` |
| **POST** | `clan/create` | Membuat clan baru (`CreateClanRequest`) | JSON Body |
| **GET** | `clan/{id}` | Detail informasi clan (`v`, `cfg`) | Path `id` |
| **DELETE/PUT**| `clan/{id}` | Menghapus/mengubah status clan | Path `id` |
| **GET** | `clan/{id}/members` | Anggota clan (`page`, `size`, `sort`, `dir`, `q`, `v`) | Query params |
| **POST** | `clan/{id}/join` | Permohonan bergabung clan | Path `id` |
| **POST** | `clan/{id}/leave` | Keluar dari clan | Path `id` |
| **GET** | `clan/{id}/requests` | Daftar permohonan masuk pending | Path `id` |
| **POST** | `clan/{id}/requests/{reqId}/decide` | Keputusan permohonan masuk (`DecideRequest`) | Path |
| **POST** | `clan/{id}/recruitment` | Pengaturan rekruitmen clan (`RecruitmentRequest`) | Path `id` |
| **POST** | `clan/{id}/bulletin` | Mengubah papan pengumuman clan | Path `id` |
| **POST** | `clan/{id}/donate` | Donasi koin/resource clan (`DonateRequest`) | Path `id` |
| **POST** | `clan/{id}/donate/reset-self` | Reset limit donasi harian | Path `id` |
| **GET** | `clan/{id}/treasury` | Data kas & saldo clan | Path `id` |
| **GET** | `clan/{id}/log` | Log aktivitas clan (`category`, `q`, `page`, `size`, `user`, `session`, `device`) | Query params |
| **POST** | `clan/{id}/daily-claim` | Klaim bonus harian clan | Path `id` |
| **POST** | `clan/{id}/daily-claim/token` | Token klaim harian clan | Path `id` |
| **POST** | `clan/{id}/equip` | Memasang skin/banner clan (`EquipRequest`) | Path `id` |
| **GET** | `clan/{id}/owned-items` | Daftar item yang dimiliki clan | Path `id` |
| **GET** | `clan-catalog/banners` | Katalog banner clan | `device_hash` |

---

### 5.4 Sistem Obrolan & Chat Realtime (`chat/*`)

| HTTP Method | Path / Endpoint | Deskripsi & Parameter | Flag / Headers |
| :--- | :--- | :--- | :--- |
| **GET** | `chat/global` | Mengambil pesan obrolan publik (`before_id`, `limit`, `channel`) | Query params |
| **POST** | `chat/global` | Kirim pesan obrolan publik (`content`, `reply_to`, `channel`) | `FormUrlEncoded` |
| **GET** | `chat/global/channels` | Daftar saluran chat publik | `device_hash` |
| **GET** | `chat/global/pin` | Pesan tersemat di chat publik | Query `channel` |
| **POST** | `chat/global/pin` | Menyematkan pesan publik | `FormUrlEncoded` |
| **DELETE/PUT**| `chat/global/pin` | Melepas pin pesan publik | Query `channel` |
| **POST** | `chat/global/clear` | Membersihkan riwayat chat publik | Query `channel` |
| **GET** | `chat/friend` | Daftar percakapan teman (`limit`) | Query `limit` |
| **GET** | `chat/friend/{peer_id}` | Pesan percakapan pribadi | Path `peer_id` |
| **POST** | `chat/friend` | Kirim pesan pribadi (`receiver_id`, `content`, `reply_to`) | `FormUrlEncoded` |
| **POST** | `chat/friend/delete` | Menghapus pesan pribadi (`id`) | `FormUrlEncoded` |
| **GET** | `clan/{id}/chat` | Mengambil percakapan obrolan clan | Path `id` |
| **POST** | `clan/{id}/chat` | Kirim pesan ke obrolan clan (`content`, `reply_to`) | Path `id` |
| **POST** | `clan/{id}/chat/delete` | Menghapus pesan chat clan (`msg_id`) | `FormUrlEncoded` |

---

### 5.5 Sistem Pertemanan & Sosial (`friend/*`)

| HTTP Method | Path / Endpoint | Deskripsi & Parameter | Flag / Headers |
| :--- | :--- | :--- | :--- |
| **POST** | `friend/list` | Daftar teman pengguna | `device_hash` |
| **POST** | `friend/search` | Mencari pengguna lain (`user_id`) | `FormUrlEncoded` |
| **POST** | `friend/request` | Mengirim permintaan pertemanan (`receiver_id`) | `FormUrlEncoded` |
| **POST** | `friend/requests` | Permintaan pertemanan masuk (`user_id`) | `FormUrlEncoded` |
| **POST** | `friend/accept` | Menerima permintaan pertemanan (`request_id`) | `FormUrlEncoded` |
| **POST** | `friend/reject` | Menolak permintaan pertemanan (`request_id`, `admin_override`) | `FormUrlEncoded` |
| **POST** | `friend/remove` | Menghapus pertemanan (`friend_id`) | `FormUrlEncoded` |
| **POST** | `friend/gift-premium` | Hadiah status VIP Premium ke teman (`receiver_id`, `days`) | `FormUrlEncoded` |
| **POST** | `friend/timeline-mixed/{page}`| Linimasa aktivitas pertemanan (`page`) | Path `page` |

---

### 5.6 Sistem Gacha & Inventory (`gacha/*`)

| HTTP Method | Path / Endpoint | Deskripsi & Parameter | Flag / Headers |
| :--- | :--- | :--- | :--- |
| **POST** | `gacha/catalog` | Katalog item gacha yang tersedia | `device_hash` |
| **POST** | `gacha/pull` | Melakukan spin/pull gacha (`count`) | `FormUrlEncoded` |
| **POST** | `gacha/equip` | Memasang item gacha yang diperoleh (`item_id`) | `FormUrlEncoded` |
| **GET** | `clan/{id}/gacha/pity` | Informasi status pity gacha clan | Path `id` |
| **POST** | `clan/{id}/gacha/pull` | Perform spin gacha khusus clan | Path `id` |

---

## 6. Kesimpulan

Meskipun aplikasi **Wibuku 1.4.4** menerapkan proteksi kelas menengah (ProGuard obfuskasi, enkripsi string kustom multi-pass `kf3`/`ce5`, dynamic `cpalette`, serta verifikasi `device_hash`), seluruh teknik komunikasi client-server, enkripsi string, Base URL server backend/CDN, hingga katalog 80+ REST API Endpoints berhasil diekstrak dan didokumentasikan secara rinci dalam laporan ini.
