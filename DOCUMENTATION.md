# 🐻 BearBridge — Dokumentasi Integrasi & Panduan Penggunaan

**BearBridge** adalah sistem *Cross-Server Command Dispatch Bridge* untuk **BearCraft Network**. Plugin ini menjembatani transaksi dari **Web Store (Payment Gateway)** langsung ke dalam server **Minecraft (Paper/Purpur)** secara otomatis, aman (*atomic lock*), dan *real-time*.

* **Lead Developer:** Yonyon (Discord: `iyonhungkul`)
* **Dedicated To:** VLADMXIV (BearCraft Network)
* **Database Host Terkunci:** `dbbb.beruang.my.id`
* **Nama Database Terkunci:** `bearcraft_store`

---

## Daftar Isi
1. [Arsitektur Alur Kerja (Workflow)](#1-arsitektur-alur-kerja-workflow)
2. [Panduan untuk Web Developer (Integrasi Web Store)](#2-panduan-untuk-web-developer-integrasi-web-store)
   - [Skema Tabel Database (`pending_commands`)](#21-skema-tabel-database-pending_commands)
   - [Cara Menyuntikkan Pesanan (INSERT Query)](#22-cara-menyuntikkan-pesanan-insert-query)
   - [Aturan Penulisan Command & Multi-Command](#23-aturan-penulisan-command--multi-command)
   - [Contoh Implementasi Kode (PHP / Node.js)](#24-contoh-implementasi-kode-php--nodejs)
   - [Memantau Status Pesanan dari Web](#25-memantau-status-pesanan-dari-web)
3. [Panduan untuk Server Developer / Admin Minecraft](#3-panduan-untuk-server-developer--admin-minecraft)
   - [Instalasi & Setup Awal (Standby Mode)](#31-instalasi--setup-awal-standby-mode)
   - [Konfigurasi Multi-Server (`target-server`)](#32-konfigurasi-multi-server-target-server)
   - [Daftar Perintah In-Game & Console (`/bb`)](#33-daftar-perintah-in-game--console-bb)
   - [Kustomisasi Pesan, Efek In-Game, & Discord Webhook](#34-kustomisasi-pesan-efek-in-game--discord-webhook)
   - [Cara Update Plugin (`/bb update`)](#35-cara-update-plugin-bb-update)

---

## 1. Arsitektur Alur Kerja (Workflow)

1. **Pembelian di Web Store:** Pemain membeli item/rank di website dan pembayaran terverifikasi (misal via Midtrans/Xendit/Tripay/Duitku).
2. **Insert ke Database:** Backend Web Store melakukan `INSERT` baris baru ke tabel `pending_commands` di database `bearcraft_store` dengan status `'PENDING'`.
3. **Deteksi Otomatis oleh BearBridge:**
   * **Jika Pemain Sedang Online:** Dalam hitungan detik (default setiap `3 detik`), BearBridge mendeteksi pesanan tersebut, mengunci statusnya menjadi `'PROCESSING'`, lalu mengeksekusi command di Console dan memunculkan efek selebrasi (Title, Sound, Broadcast, Discord Webhook).
   * **Jika Pemain Sedang Offline:** Pesanan akan tetap tersimpan aman di database dengan status `'PENDING'`. Begitu pemain tersebut masuk ke dalam server (`PlayerJoinEvent`), BearBridge otomatis langsung mengeksekusi pesanannya setelah 1 detik pemain *login*.
4. **Finalisasi Status:** Setelah dieksekusi, status pesanan otomatis berubah menjadi `'SUCCESS'` (atau `'FAILED'` beserta pesan error-nya jika ada kendala).

---

## 2. Panduan untuk Web Developer (Integrasi Web Store)

Bagian ini ditujukan bagi Web Developer yang mengelola backend toko/payment webhook BearCraft.

### 2.1. Skema Tabel Database (`pending_commands`)
Tabel `pending_commands` akan dibuat secara otomatis oleh plugin BearBridge saat pertama kali terhubung ke database `bearcraft_store`. Berikut adalah strukturnya:

```sql
CREATE TABLE IF NOT EXISTS `pending_commands` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `player_name` VARCHAR(32) NOT NULL,
    `command` TEXT NOT NULL,
    `target_server` VARCHAR(32) NOT NULL,
    `status` ENUM('PENDING', 'PROCESSING', 'SUCCESS', 'FAILED') NOT NULL DEFAULT 'PENDING',
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    `executed_at` TIMESTAMP NULL DEFAULT NULL,
    `processed_by` VARCHAR(64) NULL DEFAULT NULL,
    `error_message` VARCHAR(255) NULL DEFAULT NULL,
    INDEX `idx_server_status` (`target_server`, `status`),
    INDEX `idx_player_server` (`player_name`, `target_server`, `status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 2.2. Cara Menyuntikkan Pesanan (INSERT Query)
Saat callback/webhook pembayaran dari payment gateway menyatakan **LUNAS (PAID/SETTLEMENT)**, backend website cukup menjalankan query `INSERT` ke3 kolom utama berikut (`player_name`, `command`, `target_server`):

```sql
INSERT INTO `pending_commands` (`player_name`, `command`, `target_server`, `status`)
VALUES ('NamaPemain', 'lp user %player% parent add vip', 'survival', 'PENDING');
```

#### Penjelasan Kolom Wajib:
| Kolom | Tipe | Penjelasan |
| :--- | :--- | :--- |
| `player_name` | `VARCHAR(32)` | Username Minecraft pembeli (Case-insensitive saat pencarian online, namun disarankan sesuai input pemain). |
| `command` | `TEXT` | Perintah console yang akan dijalankan (tanpa `/` di depan, meskipun jika ada `/` otomatis dibersihkan oleh plugin). |
| `target_server` | `VARCHAR(32)` | ID server tujuan (misal: `'survival'`, `'skyblock'`, `'lobby'`). **Harus sama persis** dengan `server.target-server` di `config.yml` pada server Minecraft tujuan. |
| `status` | `ENUM` | Biarkan default `'PENDING'` saat baru dibuat. |

---

### 2.3. Aturan Penulisan Command & Multi-Command

1. **Mendukung Placeholder `%player%` atau `{player}`:**
   * Anda boleh menyuntikkan nama pemain secara langsung (`lp user Budi parent add vip`) **ATAU** menggunakan placeholder (`lp user %player% parent add vip`). Plugin akan otomatis mengganti `%player%` dan `{player}` dengan isi kolom `player_name`.
2. **Tanpa Garis Miring (`/`) di Awal:**
   * Tulis perintah sebagaimana diketik di Console (contoh: `eco give %player% 50000`). Jika terlanjur menggunakan `/` di awal (`/eco give...`), BearBridge tetap akan membersihkannya secara otomatis.
3. **Eksekusi Banyak Command Sekaligus (Multi-Command dengan `;`):**
   * Jika 1 paket pembelian di web berisi beberapa hadiah sekaligus (misal: dapat Rank + Uang + Key Crate), Anda **tidak perlu** membuat banyak baris di database. Cukup pisahkan dengan tanda titik koma (`;`) dalam satu kolom `command`:
   ```text
   lp user %player% parent add mvp; eco give %player% 100000; crate key give %player% legendary 3
   ```
4. **Deteksi Otomatis Rank (LuckPerms Integration):**
   * Jika command mengandung pola `parent add <rank>`, `parent set <rank>`, `group add <rank>`, atau `group set <rank>`, BearBridge otomatis mengenali nama rank tersebut untuk ditampilkan pada pengumuman *Broadcast*, *Console Log*, dan *Discord Webhook* (Perubahan dari Rank Lama $\rightarrow$ Rank Baru).

---

### 2.4. Contoh Implementasi Kode (Express.js / Node.js)

Berikut adalah contoh struktur siap pakai untuk backend **Express.js** menggunakan library `mysql2`:

#### 1. Instalasi Package
```bash
npm install express mysql2 dotenv
```

#### 2. Konfigurasi Database Pool (`db.js`)
```javascript
const mysql = require('mysql2/promise');
require('dotenv').config();

const bearBridgePool = mysql.createPool({
  host: 'dbbb.beruang.my.id',
  port: 3306,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: 'bearcraft_store',
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0
});

/**
 * Fungsi helper untuk mengirim command ke antrian BearBridge
 * @param {string} playerName - Username Minecraft pembeli
 * @param {string|string[]} commands - String command atau Array berisi banyak command
 * @param {string} targetServer - Server tujuan (contoh: 'survival', 'skyblock')
 */
async function dispatchToMinecraft(playerName, commands, targetServer = 'survival') {
  // Jika dikirim dalam bentuk Array, gabungkan otomatis dengan '; '
  const finalCommand = Array.isArray(commands) ? commands.join('; ') : commands;

  const [result] = await bearBridgePool.execute(
    'INSERT INTO `pending_commands` (`player_name`, `command`, `target_server`, `status`) VALUES (?, ?, ?, ?)',
    [playerName.trim(), finalCommand, targetServer.toLowerCase(), 'PENDING']
  );

  return result.insertId; // Mengembalikan Order ID di BearBridge
}

module.exports = { bearBridgePool, dispatchToMinecraft };
```

#### 3. Contoh Endpoint di Express.js (`app.js` / `routes/webhook.js`)
```javascript
const express = require('express');
const { bearBridgePool, dispatchToMinecraft } = require('./db');

const app = express();
app.use(express.json());

/**
 * 1. ENDPOINT CALLBACK PAYMENT GATEWAY (Midtrans / Tripay / Xendit / Duitku)
 * Dipanggil otomatis saat pembayaran berstatus PAID / SETTLEMENT
 */
app.post('/api/payment/callback', async (req, res) => {
  try {
    const { status, player_name, package_commands, server_mode } = req.body;

    // Pastikan hanya mengeksekusi jika pembayaran sudah LUNAS
    if (status !== 'PAID' && status !== 'SETTLEMENT') {
      return res.status(200).json({ message: 'Ignored: Payment not settled yet' });
    }

    // Contoh: package_commands bisa berupa string tunggal atau array multi-command:
    // ["lp user %player% parent add mvp", "eco give %player% 250000", "crate key give %player% mythic 2"]
    const orderId = await dispatchToMinecraft(
      player_name,
      package_commands,
      server_mode || 'survival'
    );

    return res.status(200).json({
      success: true,
      message: 'Command queued to BearBridge',
      bearbridge_order_id: orderId
    });
  } catch (error) {
    console.error('[BearBridge Webhook Error]:', error);
    return res.status(500).json({ success: false, error: 'Database insert failed' });
  }
});

/**
 * 2. ENDPOINT CEK STATUS PESANAN (Opsional untuk halaman Invoice / Status di Web)
 * GET /api/order-status/:id
 */
app.get('/api/order-status/:id', async (req, res) => {
  try {
    const [rows] = await bearBridgePool.execute(
      'SELECT `id`, `player_name`, `target_server`, `status`, `created_at`, `executed_at`, `error_message` FROM `pending_commands` WHERE `id` = ?',
      [req.params.id]
    );

    if (rows.length === 0) {
      return res.status(404).json({ success: false, message: 'Order not found' });
    }

    return res.status(200).json({
      success: true,
      data: rows[0]
    });
  } catch (error) {
    return res.status(500).json({ success: false, error: error.message });
  }
});

app.listen(3000, () => console.log('Store Backend running on port 3000'));
```

---

### 2.5. Memantau Status Pesanan dari Web
Jika halaman *Riwayat Pembelian* atau *Admin Panel* di Website ingin menampilkan status apakah item sudah masuk ke dalam game atau belum, Web Developer cukup membaca kolom berikut pada tabel `pending_commands`:
* `status = 'PENDING'` $\rightarrow$ Menunggu pemain online di server tujuan.
* `status = 'PROCESSING'` $\rightarrow$ Sedang dieksekusi di dalam server.
* `status = 'SUCCESS'` $\rightarrow$ Sudah berhasil diterima pemain pada waktu `executed_at`.
* `status = 'FAILED'` $\rightarrow$ Gagal dieksekusi (lihat penyebabnya di kolom `error_message`).

---

## 3. Panduan untuk Server Developer / Admin Minecraft

Bagian ini ditujukan bagi Developer/Admin yang memasang plugin `BearBridge.jar` di server Minecraft.

### 3.1. Instalasi & Setup Awal (Standby Mode)
1. Masukkan file **`BearBridge.jar`** ke dalam folder `plugins/` di server Paper/Purpur Anda.
2. Nyalakan server.
3. Saat pertama kali dijalankan, karena kredensial database belum diisi, plugin **tidak akan crash/mati**, melainkan masuk ke **Standby Mode** dan menampilkan instruksi di Console:
   ```text
   ------ ʙᴇᴀʀʙʀɪᴅɢᴇ ꜱʏꜱᴛᴇᴍ - ꜱᴛᴀɴᴅʙʏ ᴍᴏᴅᴇ ------
   ᴍᴏᴅᴇ: ꜱᴜʀᴠɪᴠᴀʟ  ▪  ᴅᴀᴛᴀʙᴀꜱᴇ: [ɴᴏᴛ ᴄᴏɴꜰɪɢᴜʀᴇᴅ]  ▪  ꜱᴛᴀᴛᴜꜱ: [ꜱᴛᴀɴᴅʙʏ]
   ⚠ Kredensial Database belum diatur di dalam konfigurasi!
   ▪ Buka file: plugins/BearBridge/config.yml
   ▪ Isi bagian database.username dan database.password sesuai kredensial server.
   ▪ Setelah disimpan, cukup ketik /bb reload (tanpa perlu restart server).
   ```
4. Buka file `plugins/BearBridge/config.yml`, lalu isi `username` dan `password` database Anda:
   ```yaml
   server:
     target-server: "survival"
     poll-interval-seconds: 3

   database:
     username: "user_db_anda"
     password: "password_db_anda"
     port: 3306
   ```
5. Simpan file `config.yml`, lalu ketik **`/bb reload`** di Console atau In-Game. Plugin akan langsung terhubung ke database dan mulai memproses antrian tanpa perlu restart server!

---

### 3.2. Konfigurasi Multi-Server (`target-server`)
Jika jaringan BearCraft memiliki banyak sub-server (misalnya: **Survival**, **Skyblock**, **OneBlock**):
* Pasang `BearBridge.jar` di masing-masing sub-server tersebut.
* Pada server Survival, atur `target-server: "survival"` di `config.yml`.
* Pada server Skyblock, atur `target-server: "skyblock"` di `config.yml`.
* Dengan begitu, pesanan yang ditujukan untuk Skyblock hanya akan dieksekusi saat pemain tersebut berada di server Skyblock, tidak akan salah masuk di server lain.

---

### 3.3. Daftar Perintah In-Game & Console (`/bb`)
Semua sub-command admin membutuhkan permission: **`bearbridge.admin`** (atau status OP).
Alias perintah: `/bearbridge`, `/bbridge`, `/bb`.

| Perintah | Fungsi & Kegunaan |
| :--- | :--- |
| **`/bb`** | Menampilkan panel informasi utama BearBridge, status koneksi DB, jumlah pesanan tereksekusi, dan status versi plugin. |
| **`/bb status`** | Menampilkan diagnostik langsung (*Live Diagnostics*) koneksi HikariCP (Active, Idle, Total) dan jumlah antrian `PENDING` saat ini. |
| **`/bb reload`** | Memuat ulang `config.yml`, `messages.yml`, dan `discordwebhook.yml`, serta menghubungkan ulang koneksi database & timer antrian. |
| **`/bb test <player> <cmd>`** | Menyuntikkan pesanan simulasi ke database untuk mengetes apakah alur antrian, efek in-game, broadcast, dan webhook berjalan lancar. <br>*Contoh:* `/bb test VLADMXIV eco give %player% 1000` |
| **`/bb retry <order_id>`** | Mengubah status pesanan tertentu yang `FAILED` kembali menjadi `PENDING` agar dieksekusi ulang oleh sistem. <br>*Contoh:* `/bb retry 42` |
| **`/bb retry all`** | Mengembalikan seluruh pesanan yang berstatus `FAILED` pada server node tersebut menjadi `PENDING` sekaligus. |
| **`/bb update`** | Mengunduh pembaruan terbaru `BearBridge.jar` langsung dari GitHub Releases resmi ke folder `plugins/update/` (otomatis terpasang saat server restart). |

---

### 3.4. Kustomisasi Pesan, Efek In-Game, & Discord Webhook

1. **`plugins/BearBridge/messages.yml`**
   * Mengatur pesan pribadi ke pembeli (**Chat**, **Title & Subtitle**, **Action Bar**, dan **Sound FX**) serta pengumuman **Global Broadcast** ke seluruh server.
   * Mendukung format warna Legacy (`&a`, `&b`, `&l`) dan Hex Color (`&#00F5FF`).
   * Placeholder yang didukung: `%player%`, `%order_id%`, `%rank%`, `%command%`.

2. **`plugins/BearBridge/discordwebhook.yml`**
   * Untuk mengaktifkan log transaksi ke Discord, ubah `features.discord-webhook: true` di `config.yml`, lalu masukkan URL Webhook Discord Anda pada bagian `webhook.url` di `discordwebhook.yml`.
   * Mendukung placeholder dinamis `%player%`, `%order_id%`, `%target_server%`, `%server_mode%`, `%old_rank%`, `%new_rank%`, `%command%`, dan `%error%`.
   * Secara bawaan, `avatar-url` dan `thumbnail-url` menggunakan `https://minotar.net/helm/%player%/128.png` sehingga gambar pada pesan Discord otomatis menampilkan kepala skin Minecraft pembeli.
   * Bagian `footer.text` dan `footer.icon-url` juga dapat diubah bebas sesuai keinginan server.

---

### 3.5. Cara Update Plugin (`/bb update`)
Ketika Lead Developer (**Yonyon**) merilis perbaikan bug atau fitur baru:
1. Anda akan melihat **notifikasi otomatis** di Console maupun di Chat saat Anda login ke dalam server (tanpa perlu mengecek website).
2. Cukup klik tombol **`[/bb update]`** di chat atau ketik **`/bb update`**.
3. Plugin akan mengunduh versi terbaru ke dalam folder `plugins/update/BearBridge.jar`.
4. Lakukan **Restart Server** kapan pun Anda siap, dan Paper/Purpur akan otomatis mengganti plugin lama dengan versi terbaru.
