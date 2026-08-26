# Analisis Keamanan & Arsitektur Firebase Wibuku 1.4.4

Dokumen ini berisi hasil analisis teknis mendalam (*static security analysis*) mengenai integrasi **Google Firebase Services** pada aplikasi **Wibuku v1.4.4** (`Wibuku_1.4.4.xapk`).

---

## 1. Parameter Konfigurasi Firebase (`res/values/strings.xml`)

Berdasarkan ekstraksi file sumber daya (`strings.xml`), berikut adalah parameter kunci integrasi Firebase yang tertanam di dalam APK Wibuku 1.4.4:

| Parameter Key | Nilai Eksplisit / String | Fungsi / Layanan Terkait |
| :--- | :--- | :--- |
| **`project_id`** | `wibuku-c87cc` | Firebase Project Identifier |
| **`google_api_key`** | `AIzaSyDUJjqSudrMD2JuRQiQP8alfxOPiSQ603A` | Google Client API Identifer Key |
| **`google_app_id`** | `1:1010948816147:android:9e17cf8a9be6d3e56caee6` | Firebase App Registration ID |
| **`google_storage_bucket`** | `wibuku-c87cc.appspot.com` | Google Cloud Storage Bucket Name |
| **`gcm_defaultSenderId`** | `1010948816147` | FCM Sender ID (Cloud Messaging) |
| **`crashlytics.mapping_id`**| `bd5dcb0cc6474df5a265eacbad011948` | Crashlytics De-obfuscation ID |

---

## 2. Analisis Layanan Firebase yang Digunakan

Dari penelusuran kode Smali & Manifest (`AndroidManifest.xml`), Wibuku 1.4.4 menggunakan layanan Firebase berikut:

1. **Firebase Cloud Messaging (FCM)**:
   - **Komponen**: `wibuku.app.wibuku.singleton.MessagingService` & `FirebaseMessagingService`.
   - **Fungsi**: Menerima notifikasi push real-time untuk pesan obrolan, permintaan pertemanan, dan pembaruan episode anime.
2. **Firebase Analytics & Performance Monitoring**:
   - **Komponen**: `FirebaseAnalytics` (`MainActivity.java`), `FirebasePerfRegistrar`.
   - **Fungsi**: Mengumpulkan metrik statistik penggunaan aplikasi dan performa HTTP request.
3. **Firebase Crashlytics**:
   - **Komponen**: `CrashlyticsRegistrar`.
   - **Fungsi**: Pelaporan log error dan crash aplikasi otomatis.
4. **Firebase Remote Config**:
   - **Komponen**: `FirebaseRemoteConfig`.
   - **Fungsi**: Pengaturan konfigurasi aplikasi dinamis dari cloud secara real-time.

---

## 3. Penilaian Keamanan (Firebase Security Assessment)

### 3.1 Apakah `google_api_key` Ekosistem Firebase Merupakan Celah Keamanan (*Vulnerability*)?
* **Fakta Teknis Google Firebase**:
  API Key Firebase (`AIzaSy...`) pada aplikasi Android secara desain **BUKAN merupakan secret/sandi rahasia**. API Key ini sengaja ditanam di client Android sebagai pengenal publik (*client identifier*) agar perangkat dapat berkomunikasi dengan Google Play Services.
* **Mekanisme Perlindungan**:
  Keamanan data Firebase tidak ditentukan oleh API Key, melainkan oleh **Firebase Security Rules** di sisi server Google Cloud (seperti Firestore Rules, Realtime Database Rules, dan Cloud Storage Rules).

---

### 3.2 Audit Ekosistem Cloud & Database Firebase Wibuku
* **Realtime Database / Firestore**:
  Aplikasi Wibuku **TIDAK menggunakan** Firebase Realtime Database atau Cloud Firestore sebagai basis data utama. Seluruh data transaksi, obrolan, dan katalog anime dikelola oleh **Custom REST Backend API** (`https://api.wibuku.app/`).
* **Cloud Storage Bucket (`wibuku-c87cc.appspot.com`)**:
  Storage bucket default Firebase tidak aktif untuk publik. Aset media aplikasi disimpan dan dilayani oleh CDN terpisah (`https://cdn.wibuku.app/`).

---

## 4. Rekomendasi Penguatan Keamanan Firebase (Hardening Recommendations)

1. **Batasi API Key di Google Cloud Console**:
   Atur **API Key Restrictions** pada Google Cloud Console agar API Key `AIzaSyDUJjqSudrMD2JuRQiQP8alfxOPiSQ603A` hanya dapat dipanggil oleh SHA-1 Certificate Fingerprint dan Package Name resmi `wibuku.app.wibuku`.
2. **Aktifkan Firebase App Check**:
   Integrasikan **Firebase App Check** (menggunakan Play Integrity provider) pada proyek Firebase `wibuku-c87cc` untuk memastikan request notifikasi FCM dan Remote Config hanya diterima dari APK asli yang sah.

---

## 5. Kesimpulan

Konfigurasi Firebase pada Wibuku 1.4.4 (`google_api_key`, `project_id`, `google_app_id`) merupakan parameter pengenal standar Android yang aman secara desain. Aplikasi tidak menyimpan data sensitif pada database Firebase Cloud, melainkan mengandalkan backend kustom (`api.wibuku.app`). Pengembang direkomendasikan menambahkan pembatasan SHA-1 fingerprint pada Google Cloud Console untuk mencegah penggunaan API Key oleh aplikasi lain.
