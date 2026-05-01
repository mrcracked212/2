# Telegram Share Promosi Bot

Bot Telegram untuk layanan share promosi dengan dukungan:
- target `group`, `channel`, `target` (group+channel), `user`, `all`
- paket `trial`, `pro`, `vip`
- trial otomatis berbasis valid target gabungan (group + channel)
- voucher, antrian promosi, admin panel command lengkap
- penyimpanan JSON (tanpa SQL) dengan atomic write dan write queue

---

## 1. Ringkasan Fitur

### Fitur user
- Registrasi awal via `/start`
- Opt-in / opt-out penerimaan promosi user-target
- Trial berbasis klaim group/channel valid
- Share promosi via wizard (`/buatpromo`) atau direct command (`/share...`)
- Klaim target pribadi: group/channel
- Cek profil, status paket, cooldown, dan limit harian

### Fitur admin
- Manajemen user, role, package, voucher, delay, limit
- Broadcast multi-scope (user/group/channel/target/all)
- Manajemen group & channel (cek, refresh, blacklist, remove)
- Log, backup, export data, repair state
- Konfirmasi untuk aksi berbahaya (`cleargroups`, `clearchannels`, `clearlogs`, `shutdown`)

### Keamanan & stabilitas
- Rate limit command (default 1.2 detik per user)
- Filter kata terlarang + warning + auto-ban setelah ambang warning
- Batas panjang pesan promosi
- Persistensi session wizard + expiry
- Atomic JSON write untuk mengurangi risiko corrupt data

---

## 2. Arsitektur Singkat

- Runtime: Node.js
- Telegram library: `node-telegram-bot-api`
- Entry point: `bot.js`
- Konfigurasi: `config.js`
- Storage helper: `db.js`
- Database: file JSON dalam folder `data/`

Semua data utama disimpan dalam JSON:
- user
- group
- channel
- voucher
- package
- promotion history
- logs
- pending group
- sessions
- settings

---

## 3. Persiapan & Instalasi

## 3.1 Prasyarat
- Node.js 18+ (disarankan)
- NPM
- Bot Telegram dari BotFather

## 3.2 Install dependency
```bash
npm install
```

## 3.3 Konfigurasi `config.js`
Edit file `config.js`:

```js
module.exports = {
  BOT_TOKEN: "ISI_TOKEN_BOT",
  OWNER_ID: 123456789,
  ADMINS: [123456789, 987654321],
  BOT_NAME: "Bot Jasa Share Promosi",
  COMMAND_PREFIX: "/",
  DATABASE_PATH: "./data",
  MIN_GROUP_MEMBER_FOR_TRIAL: 30,
  REQUIRED_TRIAL_GROUPS: 3,
  DEFAULT_DELAYS: { trial: 60, pro: 30, vip: 15 },
  BROADCAST: { BATCH_SIZE: 20, DELAY_BETWEEN_BATCH_MS: 3000, MAX_MESSAGE_LENGTH: 3500 },
  SECURITY: { MAX_PROMO_PER_DAY_TRIAL: 3, MAX_PROMO_PER_DAY_PRO: 20, MAX_PROMO_PER_DAY_VIP: 50 }
};
```

### Penjelasan field penting
- `BOT_TOKEN`: token bot dari BotFather (wajib)
- `OWNER_ID`: Telegram ID owner utama
- `ADMINS`: daftar Telegram ID admin tambahan
- `DATABASE_PATH`: lokasi folder JSON
- `MIN_GROUP_MEMBER_FOR_TRIAL`: batas minimum member group trial
- `REQUIRED_TRIAL_GROUPS`: default jumlah target valid untuk aktivasi trial
- `DEFAULT_DELAYS`: cooldown per paket (detik)
- `BROADCAST`: batch dan batas panjang pesan
- `SECURITY`: limit promo harian default per paket

> Catatan: admin runtime diturunkan dari `OWNER_ID` + `ADMINS` di `config.js`.

## 3.4 Jalankan validasi syntax
```bash
npm run check
```

## 3.5 Jalankan bot
```bash
npm start
```

Jika berhasil, console akan menampilkan bot aktif sebagai `@username_bot`.

---

## 4. Struktur Data

Folder data default: `data/`

File yang dipakai:
- `users.json`
- `groups.json`
- `channels.json`
- `vouchers.json`
- `packages.json`
- `promotions.json`
- `logs.json`
- `pending_groups.json`
- `sessions.json`
- `settings.json`

Bot akan membuat file default otomatis jika belum ada.

Backup admin (`/backup`) disimpan ke:
- `data/backups/backup-<timestamp>.json`

---

## 5. Alur Penggunaan User

## 5.1 Onboarding awal
1. User buka private chat dengan bot
2. Jalankan `/start`
3. Aktifkan penerimaan promosi: `/optin`
4. Cek status akun: `/profile` dan `/status`

## 5.2 Trial Basic (group + channel)
Trial aktif otomatis jika total target valid user mencapai syarat.

Syarat default:
- total target valid: `3` (gabungan group + channel)
- group valid: minimal `30` member (tanpa bot)
- channel valid: minimal `30` subscriber + bot admin + bisa post

Command yang relevan:
- `/trial` -> lihat progress dan syarat
- `/claimgroup` -> klaim group saat dijalankan di group
- `/claimgroup GROUP_ID` -> klaim dari private chat
- `/claimchannel` -> pilih channel dari inline list di private
- `/claimchannel CHANNEL_ID` -> klaim langsung via ID
- `/claimtrial` -> helper klaim cepat (group/chat context)
- `/claimtrial CHAT_ID` -> klaim target via ID dari private
- `/mygroups`, `/mychannels`, `/mytargets` -> cek target milik user

### Catatan klaim channel
Agar channel valid trial:
- bot harus ada di channel
- bot status admin
- bot punya izin post
- subscriber memenuhi minimum

## 5.3 Buat promosi

### Metode wizard
- `/buatpromo`
- isi judul
- isi body
- pilih target (`group`, `channel`, `target`, `user`, `all`)
- konfirmasi kirim

Batalkan wizard kapan saja dengan `/cancel`.

### Metode direct command
Format umum:
- `/sharegrup JUDUL|ISI`
- `/sharechannel JUDUL|ISI`
- `/sharetarget JUDUL|ISI`
- `/shareuser JUDUL|ISI`
- `/shareall JUDUL|ISI`

## 5.4 Format tombol URL (opsional)
Body promosi mendukung 1 tombol URL dengan format:

```text
button:Nama Tombol|https://contoh.com
```

Contoh body (wizard):
```text
Diskon 50% untuk member baru.
Berlaku sampai minggu ini.
button:Daftar Sekarang|https://contoh.com/daftar
```

Validasi URL hanya menerima `http://` atau `https://`.

---

## 6. Matrix Paket & Batasan

Default package behavior:

| Paket | Target group | Target channel | Target user | Target all | Daily limit | Cooldown |
|---|---:|---:|---:|---:|---:|---:|
| trial | ya | ya | tidak | tidak | 3/hari | 60s |
| pro | ya | ya | tidak | tidak | 20/hari | 30s |
| vip | ya | ya | ya | ya | 50/hari | 15s |

Keterangan:
- `target` = gabungan `group + channel`
- admin bypass limit/cooldown paket
- paket `free` tidak bisa membuat promosi

Format durasi yang diterima:
- `10s`, `15m`, `12h`, `7d`, `2w`

---

## 7. Command User

> Banyak command user bersifat private-only.
> Pengecualian utama: `/claimgroup` dan `/claimtrial` bisa dipakai sesuai konteks chat.

| Command | Fungsi |
|---|---|
| `/start` | Mulai interaksi + tampilkan menu inline |
| `/help` | Bantuan command |
| `/profile` | Profil user, paket, trial, status ban |
| `/status` | Paket aktif, expired, cooldown, limit harian, queue |
| `/trial` | Info syarat trial dan progress target valid |
| `/optin` | Izinkan menerima promosi user-target |
| `/optout` | Berhenti menerima promosi user-target |
| `/redeem KODE` | Aktivasi voucher |
| `/buatpromo` | Wizard pembuatan promosi |
| `/sharegrup JUDUL|ISI` | Promosi target group |
| `/sharechannel JUDUL|ISI` | Promosi target channel |
| `/sharetarget JUDUL|ISI` | Promosi target group+channel |
| `/shareuser JUDUL|ISI` | Promosi target user opt-in |
| `/shareall JUDUL|ISI` | Promosi target group+channel+user |
| `/mygroups` | List group yang diklaim user |
| `/mychannels` | List channel yang diklaim user |
| `/mytargets` | Ringkasan group+channel user |
| `/claimgroup [GROUP_ID]` | Klaim group untuk trial |
| `/claimchannel [CHANNEL_ID]` | Klaim channel untuk trial |
| `/claimtrial [CHAT_ID]` | Helper klaim trial by context/ID |
| `/cancel` | Batalkan flow wizard/session |

---

## 8. Command Admin

Semua command admin dibatasi:
- hanya untuk role admin/owner
- private chat only

## 8.1 Monitoring & statistik
| Command | Fungsi |
|---|---|
| `/admin` | Panel admin + tombol cepat |
| `/stats` | Statistik global user/group/channel/promo |
| `/todaystats` | Statistik promosi hari ini |
| `/botstatus` | Queue, status worker, session aktif, ukuran DB |
| `/healthcheck` | Cek health bot |
| `/users` | Ringkasan daftar user |
| `/user USER_ID` | Detail user |
| `/searchuser KEYWORD` | Cari user by id/username/nama |

## 8.2 Moderasi user & akses
| Command | Fungsi |
|---|---|
| `/ban USER_ID` | Ban user |
| `/unban USER_ID` | Unban user |
| `/warn USER_ID ALASAN` | Tambah warning |
| `/warnings USER_ID` | Lihat riwayat warning |
| `/resetwarn USER_ID` | Reset warning user |
| `/setpackage USER_ID PACKAGE DURASI` | Set paket user |
| `/setpackage USER_ID free` | Cabut paket ke free |
| `/extend USER_ID DURASI` | Perpanjang expired paket aktif |
| `/removeaccess USER_ID` | Set user jadi free |
| `/resetlimit USER_ID` | Reset quota harian promosi user |
| `/resetcooldown USER_ID` | Reset cooldown promosi user |
| `/setrole USER_ID admin\|user` | Ubah role runtime |
| `/removeadmin USER_ID` | Hapus admin runtime |
| `/noteuser USER_ID CATATAN` | Simpan catatan admin ke user |
| `/userlogs USER_ID` | Log terkait user |

> Penting: perubahan admin via `/setrole` dan `/removeadmin` bersifat runtime, tidak menulis `config.js`. Setelah restart, role admin mengikuti `OWNER_ID` + `ADMINS` di `config.js`.

## 8.3 Voucher
| Command | Fungsi |
|---|---|
| `/createvoucher PACKAGE DURASI JUMLAH` | Generate voucher massal |
| `/createcustomvoucher KODE PACKAGE DURASI` | Buat voucher dengan kode custom |
| `/vouchers` | List voucher |
| `/voucher KODE` | Detail voucher |
| `/deletevoucher KODE` | Hapus voucher |
| `/disablevoucher KODE` | Disable voucher |
| `/enablevoucher KODE` | Enable voucher |
| `/unusedvouchers` | List voucher belum dipakai |
| `/usedvouchers` | List voucher sudah dipakai |
| `/exportvouchers` | Export voucher CSV-like text |

## 8.4 Paket & policy
| Command | Fungsi |
|---|---|
| `/packages` | Lihat konfigurasi paket aktif |
| `/setdelay PACKAGE DETIK` | Atur cooldown per paket |
| `/setlimit PACKAGE JUMLAH` | Atur limit harian per paket |
| `/settrialduration DURASI` | Atur durasi trial default |
| `/setmintrialmembers JUMLAH` | Atur minimum member group trial |
| `/setmintrialsubs JUMLAH` | Atur minimum subscriber channel trial |
| `/setrequiredtrialtarget JUMLAH` | Atur jumlah target valid untuk trial |
| `/enablepackage PACKAGE` | Aktifkan paket |
| `/disablepackage PACKAGE` | Nonaktifkan paket |

## 8.5 Broadcast
| Command | Scope |
|---|---|
| `/broadcast PESAN` | user opt-in |
| `/broadcastall PESAN` | semua user non-ban yang pernah start |
| `/broadcastuser PESAN` | user |
| `/broadcastgroup PESAN` | group aktif |
| `/broadcastchannel PESAN` | channel aktif (admin+postable) |
| `/broadcasttarget PESAN` | group + channel |
| `/cancelbroadcast` | minta pembatalan broadcast berjalan |

## 8.6 Manajemen group
| Command | Fungsi |
|---|---|
| `/groups` | List semua group |
| `/activegroups` | List group aktif |
| `/inactivegroups` | List group nonaktif |
| `/group GROUP_ID` | Detail group |
| `/groupinfo GROUP_ID` | Detail group (alias) |
| `/removegroup GROUP_ID` | Hapus group dari DB |
| `/checkgroup GROUP_ID` | Validasi ulang group |
| `/refreshgroups` | Validasi ulang semua group |
| `/blacklistgroup GROUP_ID ALASAN` | Blacklist group |
| `/unblacklistgroup GROUP_ID` | Unblacklist group |
| `/addgrouplink LINK` | Simpan link group sebagai pending |
| `/cleargroups` | Hapus semua group (butuh konfirmasi) |

## 8.7 Manajemen channel
| Command | Fungsi |
|---|---|
| `/channels` | List semua channel |
| `/activechannels` | List channel aktif |
| `/inactivechannels` | List channel nonaktif |
| `/validchannels` | List channel valid trial |
| `/invalidchannels` | List channel tidak valid trial |
| `/channel CHANNEL_ID` | Detail channel |
| `/removechannel CHANNEL_ID` | Hapus channel dari DB |
| `/checkchannel CHANNEL_ID` | Validasi ulang channel |
| `/refreshchannels` | Validasi ulang semua channel |
| `/blacklistchannel CHANNEL_ID ALASAN` | Blacklist channel |
| `/unblacklistchannel CHANNEL_ID` | Unblacklist channel |
| `/clearchannels` | Hapus semua channel (butuh konfirmasi) |

## 8.8 Logs, backup, maintenance
| Command | Fungsi |
|---|---|
| `/logs` | Log terbaru |
| `/errorlogs` | Filter log error |
| `/promologs` | Filter log promosi |
| `/adminlogs` | Filter log admin |
| `/clearlogs` | Hapus semua log (butuh konfirmasi) |
| `/exportlogs` | Export log text |
| `/backup` | Snapshot state ke file backup |
| `/restore` | Placeholder (restore manual) |
| `/exportdb` | Dump seluruh state JSON |
| `/cleancache` | Bersihkan session expired |
| `/repairdb` | Normalisasi shape state |
| `/reloadconfig` | Reload runtime config dari state/settings |
| `/shutdown` | Shutdown bot aman (butuh konfirmasi) |
| `/restartinfo` | Info restart manual |

---

## 9. Menu Inline

`/start` menampilkan inline menu cepat:
- opt-in / opt-out
- profil
- paket
- trial
- redeem
- menu promosi
- target saya
- bantuan
- menu admin (khusus admin)

Callback tambahan:
- pilih target wizard promosi
- pilih channel untuk klaim channel
- konfirmasi aksi berbahaya admin

---

## 10. Aturan Trial & Validasi Target

## 10.1 Group
Valid jika:
- bot masih ada di group
- jumlah member (tanpa bot) >= minimum

## 10.2 Channel
Valid jika:
- bot admin di channel
- bot bisa post message
- subscriber >= minimum

## 10.3 Aktivasi trial otomatis
Trial aktif jika:
- user masih paket `free`
- jumlah `valid_targets` >= `required_target_count`

Saat aktif:
- paket user di-set ke `trial`
- durasi trial mengikuti `packages.trial.default_duration`

---

## 11. Moderasi Konten Promosi

Sebelum promosi masuk queue, bot cek:
- akses paket + izin target
- cooldown + limit harian (non-admin)
- panjang pesan maksimal
- kata terlarang (`settings.moderation.banned_words`)
- format tombol URL

Jika user melanggar aturan moderasi:
- warning bertambah
- jika warning >= `max_warning`, user auto-ban

Default moderasi di `data/settings.json`:
- `max_warning`: 3
- `max_promo_length`: 3500
- `max_url_buttons`: 1

---

## 12. Queue, Delivery, dan Error Handling

- Semua promosi/broadcast diproses melalui queue global
- Pengiriman dilakukan bertahap (batch)
- Error fatal saat kirim bisa menonaktifkan target otomatis, misalnya:
  - user block bot -> user di-opt-out
  - group/channel inaccessible -> ditandai nonaktif / tidak valid

---

## 13. Operasional Harian (Saran)

1. Cek status:
   - `/stats`
   - `/botstatus`
   - `/errorlogs`
2. Cek target:
   - `/activegroups`
   - `/activechannels`
3. Rapikan data berkala:
   - `/refreshgroups`
   - `/refreshchannels`
4. Backup berkala:
   - `/backup`

---

## 14. Troubleshooting

## Bot tidak bisa start
- Gejala: `Silakan isi BOT_TOKEN di config.js...`
- Solusi: isi `BOT_TOKEN` valid di `config.js`.

## Command ditolak "hanya private chat"
- Jalankan command tersebut dari private chat bot.
- Khusus klaim group/channel, ikuti konteks command masing-masing.

## Klaim channel gagal / tidak valid
Periksa:
- bot sudah ada di channel
- bot admin
- bot punya izin post
- subscriber cukup

## Promo ditolak
Cek kemungkinan:
- paket user masih `free`
- target tidak diizinkan oleh paket
- cooldown belum selesai
- limit harian tercapai
- konten melanggar moderasi

## Broadcast salah sasaran
Pastikan command scope benar:
- `/broadcast` -> opt-in user
- `/broadcasttarget` -> group + channel
- `/broadcastall` -> semua user yang pernah start

## Role admin hilang setelah restart
- Pastikan ID admin ada di `config.js -> ADMINS`
- command `/setrole` bersifat runtime, bukan edit file config

---

## 15. Checklist Uji Cepat (1x run)

1. `npm run check` -> tidak ada syntax error
2. `npm start`
3. `/start`, `/optin`, `/status`
4. tambah bot ke group, jalankan `/claimgroup`
5. tambah bot ke channel (admin), jalankan `/claimchannel`
6. cek `/trial`, `/mytargets`
7. kirim `/buatpromo` sampai selesai
8. cek admin `/stats`, `/botstatus`, `/logs`
9. jalankan `/backup`

---

## 16. Catatan Kepatuhan

Gunakan bot ini secara legal dan sesuai kebijakan Telegram:
- jangan spam
- hormati persetujuan user (opt-in)
- hindari konten terlarang

Project ini dirancang sebagai sistem promosi terkelola (bukan spammer massal ilegal).
