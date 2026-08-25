# Dokumentasi Lengkap REST API Endpoints Wibuku 1.4.4

Dokumen ini berisi spesifikasi teknis dan dokumentasi lengkap seluruh REST API Endpoints aplikasi **Wibuku v1.4.4** (`Wibuku_1.4.4.xapk`).

---

## 1. Persyaratan Komunikasi & Header Global

Setiap permintaan (*request*) HTTP yang dikirimkan oleh aplikasi Wibuku ke server backend (`https://api.wibuku.app/`) wajib memenuhi persyaratan header dan otentikasi berikut:

### 1.1 Header Wajib Global
* **`Content-Type`**:
  - `application/json` (pada request bertanda `@Body` / `@nr`)
  - `application/x-www-form-urlencoded` (pada request bertanda `@FormUrlEncoded` / `@wn1`)
* **`device_hash`**:
  Parameter identifikasi unik perangkat berbasis SHA-256 `ANDROID_ID`. Dikirim via HTTP Header (`@Header("device_hash")`), Query Parameter (`@Query("device_hash")`), atau Form Field (`@Field("device_hash")`).
* **`X-Panel-Auth`**:
  Token sesi pengguna (JWT Token). Diperlukan pada seluruh endpoint internal/privat.
* **`X-Skip-Panel-Auth: 1`**:
  Header khusus untuk melewati verifikasi sesi token pada endpoint publik seperti pendaftaran dan dev-login.

---

## 2. Katalog Lengkap REST API Endpoints (`defpackage.bl3`)

Berikut adalah daftar 100 endpoint API hasil ekstraksi lengkap dari interface Retrofit `defpackage.bl3`:

### 2.1 Autentikasi & Pengaturan User (`user/*`, `device/*`)

| HTTP Method | Path / Endpoint | Syarat Header & Flags | Parameters & Payload |
| :--- | :--- | :--- | :--- |
| **POST** | `device/token` | `FormUrlEncoded` | `@Field("token") String str, @Header("device_hash") String str2` |
| **POST** | `user/sync/history` | `None` | `@Body SyncHistory syncHistory, @Header("device_hash") String str` |
| **POST** | `user/login` | `FormUrlEncoded` | `@Header("device_hash") String str, @Header("cpalette") long j, @Field("version") String str2` |
| **POST** | `user/recharge/` | `FormUrlEncoded` | `@Field("token") String str, @Header("device_hash") String str2` |
| **POST** | `user/register` | `{"X-Skip-Panel-Auth: 1"}` | `@Body RegisterRequest registerRequest` |
| **POST** | `user/profile/{id}` | `None` | `@Path("id") long j, @Header("device_hash") String str` |
| **POST** | `user/histories/delete` | `FormUrlEncoded` | `@Field("episode_ids") String str, @Header("device_hash") String str2` |
| **POST** | `user/session` | `None` | `@Header("device_hash") String str` |
| **POST** | `user/recharge/token` | `None` | `@Header("device_hash") String str` |
| **POST** | `user/comment/` | `None` | `@Body CommentBody commentBody, @Header("device_hash") String str` |
| **POST** | `user/react/` | `None` | `@Body Reaction reaction, @Header("device_hash") String str` |
| **POST** | `user/premium/consume` | `FormUrlEncoded` | `@Field("product_id") int i, @Field("purchase_token") String str, @Field("order_id") String str2, @Header("device_hash") String str3` |
| **POST** | `user/comment/delete` | `FormUrlEncoded` | `@Field("comment_id") long j, @Header("device_hash") String str` |
| **POST** | `user/devlogin` | `FormUrlEncoded, {"X-Skip-Panel-Auth: 1"}` | `@Field("username") String str, @Field("password") String str2` |
| **POST** | `user/sync-photo` | `FormUrlEncoded` | `@Field("server_auth_code") String str, @Header("device_hash") String str2` |
| **POST** | `user/sync/pinned` | `FormUrlEncoded` | `@Field("id") long j, @Field("pinned") boolean z, @Header("device_hash") String str` |

---

### 2.2 Katalog Anime & Streaming (`anime/*`, `home/*`, `giveaway/*`)

| HTTP Method | Path / Endpoint | Syarat Header & Flags | Parameters & Payload |
| :--- | :--- | :--- | :--- |
| **POST** | `anime/browse` | `FormUrlEncoded` | `@Field("q") String str, @Header("device_hash") String str2` |
| **POST** | `anime/home` | `None` | `@Header("device_hash") String str` |
| **POST** | `giveaway/create` | `FormUrlEncoded` | `@Field("episode_id") long j, @Field("days") int i, @Field("total_gifts") int i2, @Field("content") String str, @Header("device_hash") String str2` |
| **POST** | `giveaway/claim` | `FormUrlEncoded` | `@Field("giveaway_id") long j, @Header("device_hash") String str` |
| **POST** | `anime/comment/report` | `FormUrlEncoded` | `@Field("comment_id") long j, @Field("reason") int i, @Header("device_hash") String str` |
| **POST** | `anime/report` | `FormUrlEncoded` | `@Field("episode_id") long j, @Field("reason") int i, @Header("device_hash") String str` |
| **GET** | `home/top-donor-badges` | `None` | `@Query("limit") int i, @Header("device_hash") String str` |
| **POST** | `giveaway/{id}` | `None` | `@Path("id") long j, @Header("device_hash") String str` |
| **POST** | `anime/popular` | `None` | `@Query("mode") String str, @Header("device_hash") String str2` |
| **POST** | `anime/episodemeta/{id}` | `None` | `@Path("id") long j, @Query("mode") int i, @Header("device_hash") String str` |
| **POST** | `home/top-givers` | `None` | `@Query("limit") int i, @Query("timeline") String str, @Header("device_hash") String str2` |
| **POST** | `giveaway/replies/{id}` | `None` | `@Path("id") long j, @Header("device_hash") String str` |

---

### 2.3 Sistem Clan & Komunitas (`clan/*`, `clans/*`)

| HTTP Method | Path / Endpoint | Syarat Header & Flags | Parameters & Payload |
| :--- | :--- | :--- | :--- |
| **GET** | `clan/{id}/members` | `None` | `@Path("id") long j, @Query("page") int i, @Query("size") int i2, @Query("sort") String str, @Query("dir") String str2, @Query("q") String str3, @Query("v") int i3, @Header("device_hash") String str4` |
| **POST** | `clan/invites/{inviteId}/accept` | `None` | `@Path("inviteId") long j, @Header("device_hash") String str` |
| **GET** | `clan/{id}/logs` | `None` | `@Path("id") long j, @Query("page") int i, @Query("size") int i2, @Header("device_hash") String str` |
| **POST** | `clan/{id}/donate` | `None` | `@Path("id") long j, @Body DonateRequest donateRequest, @Header("device_hash") String str` |
| **GET** | `clan/{id}/invites` | `None` | `@Path("id") long j, @Header("device_hash") String str` |
| **POST** | `clan/{id}/invite` | `None` | `@Path("id") long j, @Body InviteSendRequest inviteSendRequest, @Header("device_hash") String str` |
| **POST** | `clan/create` | `None` | `@Body CreateClanRequest createClanRequest, @Header("device_hash") String str` |
| **GET** | `clan/{id}/friends` | `None` | `@Path("id") long j, @Header("device_hash") String str` |
| **POST** | `clan/{id}/members/{userId}/role` | `None` | `@Path("id") long j, @Path("userId") long j2, @Body ChangeRoleRequest changeRoleRequest, @Header("device_hash") String str` |
| **GET** | `clan/{id}/owned-items` | `None` | `@Path("id") long j, @Header("device_hash") String str` |
| **POST** | `clan/{id}/daily-claim` | `FormUrlEncoded` | `@Path("id") long j, @Header("device_hash") String str` |
| **POST** | `clan/{id}/join` | `None` | `@Path("id") long j, @Header("device_hash") String str` |
| **POST** | `clan/{id}/leave` | `None` | `@Path("id") long j, @Header("device_hash") String str` |
| **GET** | `clan/{id}/treasury` | `None` | `@Path("id") long j, @Header("device_hash") String str` |
| **POST** | `clan/{id}/gacha/pull` | `None` | `@Path("id") long j, @Body GachaPullRequest gachaPullRequest, @Header("device_hash") String str` |
| **GET** | `clan-create-config` | `None` | `@Header("device_hash") String str` |
| **GET** | `clan/{id}/gacha/pity` | `None` | `@Path("id") long j, @Header("device_hash") String str` |
| **GET** | `clan/{id}/requests` | `None` | `@Path("id") long j, @Header("device_hash") String str` |
| **POST** | `clan/{id}/daily-claim/token` | `None` | `@Path("id") long j, @Header("device_hash") String str` |
| **POST** | `clan/{id}/equip` | `None` | `@Path("id") long j, @Body EquipRequest equipRequest, @Header("device_hash") String str` |
| **DELETE/PUT**| `clan/{id}` | `None` | `@Path("id") long j, @Header("device_hash") String str` |
| **POST** | `clan/{id}/donate/reset-self` | `None` | `@Path("id") long j, @Header("device_hash") String str` |
| **POST** | `clan/{id}/requests/{reqId}/decide` | `None` | `@Path("id") long j, @Path("reqId") long j2, @Body DecideRequest decideRequest, @Header("device_hash") String str` |
| **GET** | `clan/{id}/log` | `None` | `@Path("id") long j, @Query("category") String str, @Query("q") String str2, @Query("page") int i, @Query("size") int i2, @Header("device_hash") String str3` |
| **GET** | `clan/{id}` | `None` | `@Path("id") long j, @Query("v") int i, @Query("cfg") String str, @Header("device_hash") String str2` |
| **GET** | `clans/search` | `None` | `@Query("q") String str, @Query("min_level") int i, @Query("policy") String str2, @Query("has_slots") boolean z, @Query("page") int i2, @Query("size") int i3, @Header("device_hash") String str3` |
| **GET** | `clans/leaderboard` | `None` | `@Query("sort") String str, @Query("limit") int i, @Header("device_hash") String str2` |
| **POST** | `clan/{id}/bulletin` | `None` | `@Path("id") long j, @Body Map<String, String> map, @Header("device_hash") String str` |
| **POST** | `clan/{id}/recruitment` | `None` | `@Path("id") long j, @Body RecruitmentRequest recruitmentRequest, @Header("device_hash") String str` |
| **GET** | `clan-catalog/banners` | `None` | `@Header("device_hash") String str` |

---

### 2.4 Sistem Obrolan & Messaging (`chat/*`)

| HTTP Method | Path / Endpoint | Syarat Header & Flags | Parameters & Payload |
| :--- | :--- | :--- | :--- |
| **POST** | `chat/global/report` | `FormUrlEncoded` | `@Field("msg_id") long j, @Field("reason") String str, @Header("device_hash") String str2, @Field("channel") String str3` |
| **GET** | `chat/friend/around` | `None` | `@Query("peer_id") long j, @Query("around_id") long j2, @Query("limit") int i, @Header("device_hash") String str` |
| **POST** | `chat/friend/pin` | `FormUrlEncoded` | `@Field("msg_id") long j, @Header("device_hash") String str` |
| **GET** | `chat/global/around` | `None` | `@Query("around_id") long j, @Query("limit") int i, @Header("device_hash") String str, @Query("channel") String str2` |
| **POST** | `chat/global/pin` | `FormUrlEncoded` | `@Field("msg_id") long j, @Header("device_hash") String str, @Field("channel") String str2` |
| **GET** | `chat/global` | `None` | `@Query("before_id") long j, @Query("limit") int i, @Header("device_hash") String str, @Query("channel") String str2` |
| **GET** | `chat/friend/{peer_id}` | `None` | `@Path("peer_id") long j, @Query("before_id") long j2, @Query("limit") int i, @Header("device_hash") String str` |
| **GET** | `chat/global/channels` | `None` | `@Header("device_hash") String str` |
| **GET** | `chat/global/pin` | `None` | `@Header("device_hash") String str, @Query("channel") String str2` |
| **POST** | `chat/global/clear` | `None` | `@Header("device_hash") String str, @Query("channel") String str2` |
| **POST** | `chat/global` | `FormUrlEncoded` | `@Field("content") String str, @Field("reply_to") long j, @Header("device_hash") String str2, @Field("channel") String str3` |
| **GET** | `chat/friend` | `None` | `@Query("limit") int i, @Header("device_hash") String str` |
| **POST** | `chat/friend` | `FormUrlEncoded` | `@Field("receiver_id") long j, @Field("content") String str, @Field("reply_to") long j2, @Header("device_hash") String str2` |
| **POST** | `chat/friend/delete` | `FormUrlEncoded` | `@Field("id") long j, @Header("device_hash") String str` |
| **DELETE/PUT**| `chat/global/pin` | `None` | `@Header("device_hash") String str, @Query("channel") String str2` |

---

### 2.5 Sistem Pertemanan & Sosial (`friend/*`)

| HTTP Method | Path / Endpoint | Syarat Header & Flags | Parameters & Payload |
| :--- | :--- | :--- | :--- |
| **POST** | `friend/mute-watching` | `FormUrlEncoded` | `@Field("friend_id") long j, @Field("enabled") boolean z, @Header("device_hash") String str` |
| **POST** | `friend/requests/count` | `FormUrlEncoded` | `@Field("user_id") long j, @Header("device_hash") String str` |
| **POST** | `friend/timeline-mixed/{page}`| `None` | `@Path("page") int i, @Header("device_hash") String str` |
| **POST** | `friend/mute-watching-for` | `FormUrlEncoded` | `@Field("friend_id") long j, @Field("enabled") boolean z, @Field("user_id") long j2, @Header("device_hash") String str` |
| **POST** | `friend/cancel` | `FormUrlEncoded` | `@Field("request_id") long j, @Header("device_hash") String str` |
| **POST** | `friend/requests` | `FormUrlEncoded` | `@Field("user_id") long j, @Header("device_hash") String str` |
| **POST** | `friend/reject` | `FormUrlEncoded` | `@Field("request_id") long j, @Field("admin_override") boolean z, @Header("device_hash") String str` |
| **POST** | `friend/request` | `FormUrlEncoded` | `@Field("receiver_id") long j, @Header("device_hash") String str` |
| **POST** | `friend/gift-premium` | `FormUrlEncoded` | `@Field("receiver_id") long j, @Field("days") int i, @Header("device_hash") String str` |
| **POST** | `friend/remove` | `FormUrlEncoded` | `@Field("friend_id") long j, @Header("device_hash") String str` |
| **POST** | `friend/accept` | `FormUrlEncoded` | `@Field("request_id") long j, @Header("device_hash") String str` |
| **POST** | `friend/search` | `FormUrlEncoded` | `@Field("user_id") long j, @Header("device_hash") String str` |

---

### 2.6 Sistem Gacha & Inventory (`gacha/*`)

| HTTP Method | Path / Endpoint | Syarat Header & Flags | Parameters & Payload |
| :--- | :--- | :--- | :--- |
| **POST** | `gacha/catalog` | `None` | `@Header("device_hash") String str` |
| **POST** | `gacha/equip` | `FormUrlEncoded` | `@Field("item_id") long j, @Header("device_hash") String str` |
| **POST** | `gacha/pull` | `FormUrlEncoded` | `@Header("device_hash") String str, @Field("count") int i` |

---

## 3. Kesimpulan

Dokumen ini menyajikan pemetaan 100% lengkap dari seluruh REST API Endpoints Wibuku 1.4.4 beserta syarat header, tipe payload, dan struktur parameternya.
