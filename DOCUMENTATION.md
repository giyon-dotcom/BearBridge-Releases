# BearBridge - Catetan Integrasi (Web sama Server)

Catetan buat integrasi backend webstore (Node.js / Express 5) ke server menkrep pake plugin **BearBridge**.
Ini bukan tipe plugin plug and play yang tinggal taro langsung jalan sendiri tanpa backend ya, jadi pastiin dari sisi web sama database udah disiapin sama alur di bawah.

Info DB yang udah ku lock di dalem plugin:
* Host DB: `dbbb.beruang.my.id` (MariaDB / MySQL, pastiin port DB nya ke open biar bisa diakses dari luar / cPanel sama server menkrep)
* Nama Database: `bearcraft_store`
* Tabel Antrian: `pending_commands` (tabelnya otomatis ke create pas plugin pertama kali konek ke DB)

---

## 1. Buat Web Dev (Backend Express.js + Duitku)

Sistemnya searah buat trigger command dari webstore ke menkrep lewat tabel antrian database (decoupled). Jadi backend Express di cPanel ga perlu ribet nembak RCON atau buka port tambahan di tiap sub server menkrep.

Alurnya simpel:
1. Player bayar via Duitku.
2. Duitku nembak endpoint callback di backend Express lu.
3. Backend validasi signature terus cek `resultCode === '00'` (pastiin ada if condition biar ga tembus / double insert).
4. Kalau lunas, backend tinggal `INSERT` 1 baris ke tabel `pending_commands` pake status `'PENDING'`.
5. Sisanya plugin BearBridge di dalem server menkrep yang bakal narik data itu terus eksekusi command nya di console.

### Struktur Tabel `pending_commands`
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

### Aturan Kolom pas `INSERT` dari Web
* **`player_name`**: Nickname menkrep player yang beli.
* **`command`**: Command console yang mau dijalanin (ga usah pake `/` di depan).
  * Bisa langsung masukin nick player nya, atau tulis `%player%` (nanti di replace otomatis sama plugin).
  * Kalau 1 paket pembelian isinya banyak command (misal dapet rank + eco + crate key), gabungin aja pake titik koma (`;`):
    ```text
    lp user %player% parent add mvp; eco give %player% 250000; crate key give %player% mythic 2
    ```
* **`target_server`**: Nama gamemode tujuan (contoh: `survival`, `skyblock`, `war`).
  * **Penting:** Karena tiap gamemode rank nya beda, string ini wajib sama persis kayak settingan `target-server` di `config.yml` plugin pada sub server tersebut.
* **`status`**: Isi `'PENDING'`.
  * Kalau anaknya lagi online di gamemode itu, paling ~3 detik langsung masuk.
  * Kalau anaknya lagi offline, datanya aman nunggu di DB, begitu dia join server langsung otomatis ke claim.

---

### Contoh Code di Express.js (Node.js 22+)

```bash
npm install mysql2
```

**`database/bearbridge.js`**
```javascript
import mysql from 'mysql2/promise';

export const bridgePool = mysql.createPool({
  host: 'dbbb.beruang.my.id',
  port: 3306,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: 'bearcraft_store',
  waitForConnections: true,
  connectionLimit: 10
});

export async function pushOrderToBridge(playerName, commands, targetServer = 'survival') {
  if (!playerName || !commands) {
    throw new Error('playerName sama commands wajib diisi');
  }

  // Kalau dikirim array, otomatis digabung pake ';'
  const cmdPayload = Array.isArray(commands) ? commands.join('; ') : commands;

  const [result] = await bridgePool.execute(
    'INSERT INTO `pending_commands` (`player_name`, `command`, `target_server`, `status`) VALUES (?, ?, ?, ?)',
    [playerName.trim(), cmdPayload, targetServer.toLowerCase(), 'PENDING']
  );

  return result.insertId;
}
```

**`routes/payment.js` (Contoh Handler Callback Duitku)**
```javascript
import express from 'express';
import { bridgePool, pushOrderToBridge } from '../database/bearbridge.js';

const router = express.Router();

router.post('/duitku/callback', async (req, res) => {
  try {
    const { resultCode, merchantOrderId, player_name, commands, gamemode } = req.body;

    // Pastiin ada if condition validasi transaksi sukses ('00') + cek signature Duitku lu
    if (resultCode !== '00') {
      return res.status(200).send('OK');
    }

    // Contoh commands: "lp user %player% parent add vip; eco give %player% 50000"
    // Contoh gamemode: "survival" / "skyblock" / "war"
    const orderId = await pushOrderToBridge(player_name, commands, gamemode);

    return res.status(200).json({ status: 'queued', bridge_order_id: orderId });
  } catch (err) {
    console.error('[BearBridge Error]:', err);
    return res.status(500).json({ error: 'Database insert error' });
  }
});

// Kalau di web mau nampilin status order udah masuk ke in game atau belum
router.get('/order-status/:id', async (req, res) => {
  try {
    const [rows] = await bridgePool.execute(
      'SELECT `id`, `player_name`, `target_server`, `status`, `created_at`, `executed_at`, `error_message` FROM `pending_commands` WHERE `id` = ?',
      [req.params.id]
    );

    if (!rows.length) return res.status(404).json({ error: 'Order not found' });
    return res.json(rows[0]);
  } catch (err) {
    return res.status(500).json({ error: err.message });
  }
});

export default router;
```

---

## 2. Buat Setup di Server Menkrep (Vlad)

### Cara Pasang sama Konfigurasi
1. Taro `BearBridge.jar` di folder `plugins/` tiap sub server (Survival, Skyblock, War, dst), tinggal idupin server.
2. Pas pertama kali idup, karena user/pass DB belum lu isi, pluginnya **ga bakal matiin diri/error**, tapi masuk ke **Standby Mode** (ada infonya di console).
3. Buka `plugins/BearBridge/config.yml`:
   ```yaml
   server:
     # Sesuaikan sama nama sub servernya (survival / skyblock / war)
     target-server: "survival"
     poll-interval-seconds: 3

   database:
     username: "user_db_lu"
     password: "password_db_lu"
     port: 3306
   ```
4. Abis di save, ga perlu restart server, langsung ketik **`/bb reload`** di console atau in game. Kalau user/pass DB bener, dia langsung konek terus mulai narik antrian.

*(Catetan: Walaupun tiap gamemode punya database sendiri buat data in game nya, khusus buat BearBridge ini semuanya konekin ke satu database `bearcraft_store` yang sama. Nanti pemisahan item/rank tiap servernya udah otomatis kefilter dari `target-server`).*

### Command Admin (`/bb`)
Permission: `bearbridge.admin` (atau OP).

* `/bb` — Cek status plugin, koneksi DB, mode server, sama versi.
* `/bb status` — Cek jumlah koneksi pool DB terus berapa antrian yang masih `PENDING`.
* `/bb reload` — Reload semua file config (`config.yml`, `messages.yml`, `discordwebhook.yml`) + reconnect DB.
* `/bb test <player> <command>` — Buat ngetes simulasi order masuk dari in game tanpa harus bayar lewat web.
  Contoh: `/bb test VLADMXIV lp user %player% parent add vip`
* `/bb retry <order_id>` — Kalau ada order yang statusnya `FAILED`, pake ini buat balikin ke `PENDING` biar dijalanin ulang.
* `/bb retry all` — Balikin semua order `FAILED` di gamemode itu jadi `PENDING`.
* `/bb update` — Gw udah pasangin fitur cek update juga. Jadi kalau sewaktu waktu ada fix bug atau update dari sisi plugin, bakal muncul notif di console terus di chat pas lu login. Tinggal ketik `/bb update`, nanti jar barunya kedownload ke `plugins/update/` terus otomatis keganti pas server restart.

### File Config Lainnya
* **`messages.yml`**: Buat ganti ganti teks notif ke player (chat, title layar, actionbar, sound) sama pengumuman broadcast satu server.
* **`discordwebhook.yml`**: Kalau mau ada log transaksi masuk ke channel Discord, ubah `discord-webhook: true` di `config.yml`, tinggal tempel URL webhook Discord nya di `discordwebhook.yml`.
