# EconomyBridge

Plugin Endstone (Python) yang menyediakan **sinkronisasi dua arah**:

```
JWEconomy (Coin)  <--->  uMoney (Money)
```

Sisi mana pun yang berubah sejak sinkronisasi terakhir yang terkonfirmasi
akan dipush ke sisi lainnya — termasuk pengurangan (mis. belanja di shop).
Jika **kedua** sisi berubah secara independen dalam siklus yang sama
(konflik asli), selisih keduanya digabung secara aditif, bukan memilih satu
"pemenang" secara sepihak. Lihat "Aturan Resolusi Konflik" di bawah untuk
detailnya.

## Kenapa Python, Bukan Java?

Endstone **tidak mendukung plugin Java** — hanya Python dan C++. Proyek ini
dibangun sebagai plugin Python resmi Endstone, tetap mengikuti arsitektur
SOLID (abstraksi provider, class dengan tanggung jawab tunggal) sesuai
permintaan spesifikasi awal.

## Struktur Proyek

```
economy-bridge/
├── pyproject.toml
├── README.md
└── src/endstone_economy_bridge/
    ├── __init__.py
    ├── plugin.py                     # EconomyBridgePlugin (class utama)
    ├── config_manager.py             # ConfigManager
    ├── bridge_manager.py             # BridgeManager (logika sinkronisasi)
    ├── sync_task.py                  # SyncTask (scheduler berkala)
    ├── sync_state_store.py           # SyncStateStore (baseline saldo terakhir per player)
    ├── economy_provider.py           # Abstraksi provider
    ├── resources/config.yml          # Default config yang dibundel
    └── providers/
        ├── jweconomy_provider.py     # Adapter JWEconomy (async)
        └── umoney_provider.py        # Adapter uMoney (sync)
```

## Tanggung Jawab Tiap Class

| Class | Tanggung Jawab |
|---|---|
| `EconomyBridgePlugin` | Entry point plugin: menyusun (wiring) komponen, menangani lifecycle (`on_load`/`on_enable`/`on_disable`), dan command `/ecobridge`. |
| `ConfigManager` | Membaca/menulis `config.yml` (bukan `config.toml` bawaan Endstone), dengan default yang aman. |
| `EconomyProvider` (`AsyncEconomyProvider` / `SyncEconomyProvider`) | Kontrak abstrak agar provider ekonomi baru mudah ditambahkan tanpa mengubah `BridgeManager`. Dipisah berdasarkan cara pemanggilan (async vs sync), bukan berdasarkan arah — kedua sisi sekarang sama-sama baca DAN tulis. |
| `JWEconomyProvider` | Adapter ke JWEconomy. API-nya **async**, dipanggil lewat `run_async()` milik JWEconomy sendiri. Baca & tulis. |
| `UMoneyProvider` | Adapter ke uMoney. API-nya sinkron biasa, baca & tulis (`api_get_player_money` / `api_reset_player_money`). |
| `BridgeManager` | Logika inti: mendengarkan `PlayerJoinEvent`, membandingkan saldo terhadap baseline terakhir, mendorong sisi mana pun yang benar-benar berubah, dan menggabungkan (merge) konflik simultan secara aditif (lihat di bawah). |
| `SyncStateStore` | Menyimpan saldo terakhir yang terkonfirmasi sinkron per player (`sync_state.json` di data folder), sehingga perubahan asli (termasuk pengurangan, mis. belanja) bisa dibedakan dari konflik asli. |
| `SyncTask` | Menjadwalkan `BridgeManager.sync_all_online()` secara berkala lewat `server.scheduler`, interval dibaca dari `config.yml`. |

## Aturan Resolusi Konflik (Sinkronisasi Dua Arah)

Karena baik JWEconomy maupun uMoney tidak mengekspos event perubahan saldo,
EconomyBridge tidak bisa langsung tahu "siapa yang berubah duluan". Sebagai
gantinya, plugin menyimpan baseline lokal kecil (`SyncStateStore`) berisi
saldo terakhir yang diketahui sudah sama di kedua sisi, dan membandingkannya
setiap siklus:

| Situasi | Artinya | Aksi |
|---|---|---|
| Coin beda dari baseline, Money tidak | Coin benar-benar berubah (mis. command, belanja shop, atau reward — naik **atau turun**) | Push Coin → uMoney |
| Money beda dari baseline, Coin tidak | Money benar-benar berubah | Push Money → JWEconomy |
| **Keduanya** beda dari baseline | Konflik asli — kedua sisi berubah independen dalam siklus yang sama | **Gabung secara aditif**: `final = baseline + Δcoin + Δmoney`, di-clamp minimal 0 |
| Tidak ada yang beda | Sudah sinkron | Tidak ada write |

> ⚠️ **Kenapa tidak pakai "saldo tertinggi menang" saja untuk konflik?**
> Tie-break naif yang cuma memilih nilai lebih besar akan membuang
> perubahan yang terjadi di sisi *lainnya*. Contoh: baseline 1000, shop
> memotong Coin jadi 800 sementara reward quest menaikkan Money jadi 1200
> dalam siklus yang sama — "tertinggi menang" akan memilih 1200 dan
> diam-diam menghapus efek belanja tadi. Dengan memperlakukan kedua
> perubahan sebagai transaksi independen yang sah dan menjumlahkan
> selisihnya ke baseline (`1000 + (-200) + (+200) = 1000`), efek **kedua**
> transaksi tetap terjaga — ide yang sama seperti menggabungkan dua
> perubahan konkuren pada sebuah counter bersama dari titik awal yang sama
> (mirip merge di sistem CRDT). Hasil gabungan di-clamp ke 0 supaya dua
> pengurangan besar yang terjadi bersamaan tidak pernah membuat saldo jadi
> negatif.

Pada rekonsiliasi pertama seorang player (belum ada baseline tercatat, mis.
tepat setelah plugin baru dipasang), tidak ada baseline untuk menghitung
selisih, sehingga satu kali itu saja jatuh ke "saldo tertinggi menang"
semata-mata untuk menetapkan baseline awal.

## Kenapa Tidak Ada Event "Coin Berubah" secara Real-time?

Berdasarkan source code JWEconomy saat ini (`junggamyeon/JWEconomy`), plugin
tersebut **tidak mengekspos event** saat saldo berubah (baik lewat command,
belanja shop, reward quest, maupun reward battle). Karena itu, mekanisme
yang realistis dan robust adalah:

1. **Sync saat join** (`sync-on-join: true`) — menutup celah saat player
   baru masuk.
2. **Scheduler berkala** (`sync-interval`, dalam detik) — menutup semua
   kasus lain (command add/remove coin, shop, reward, dll) dalam rentang
   waktu maksimal sebesar interval tersebut.

Jika di masa depan JWEconomy menambahkan event perubahan saldo, cukup
tambahkan `@event_handler` baru di `BridgeManager` yang memanggil
`sync_player()` — tidak perlu mengubah komponen lain.

## Memilih `sync-interval` yang Optimal

Tidak ada angka yang cocok untuk semua server — ini trade-off antara beban
pada database JWEconomy dan seberapa cepat perubahan saldo terlihat di
uMoney.

| Ukuran server (player online bersamaan) | `sync-interval` disarankan |
|---|---|
| Kecil (< 20 player) | 10–15 detik |
| Menengah (20–100 player) | 20–30 detik |
| Besar (100+ player) | 45–60 detik (default) |

- Terlalu kecil → scheduler memanggil `fetch_balance()` untuk **semua**
  player online tiap siklus, membebani database JWEconomy meski banyak
  yang saldonya tidak berubah (kode sudah skip *penulisan* jika sama, tapi
  *pembacaan*-nya tetap jalan).
- Terlalu besar → perubahan saldo (shop, reward) bisa telat terlihat di
  uMoney hingga durasi interval tersebut.
- Kalau plugin shop/quest/battle kamu sendiri mengekspos event (mis.
  `ShopPurchaseEvent`), lebih baik tambahkan listener khusus di
  `BridgeManager` yang memanggil `sync_player()` saat event itu terjadi —
  sehingga `sync-interval` besar tetap aman dipakai sebagai backup saja,
  sementara transaksi besar tetap terasa instan.

## Command

- `/ecobridge sync` — paksa sinkronisasi semua player online sekarang.
- `/ecobridge reload` — muat ulang `config.yml` & restart scheduler.
- `/ecobridge status` — tampilkan status konfigurasi saat ini.

Usage terdaftar sebagai `/ecobridge (sync|reload|status)<action: EcoBridgeAction>`,
mengikuti sintaks **user-defined enum type** Endstone: daftar pilihan
ditulis di dalam `(...)`, diikuti nama parameter & tipe di dalam `<...>`.

Permission: `economy_bridge.command.ecobridge` (default: **op**).

## Instalasi

```bash
pip install build
cd economy-bridge
python -m build --wheel
```

Salin file `.whl` hasil build (di folder `dist/`) ke folder `plugins/` server
Endstone kamu, lalu jalankan server. `config.yml` akan otomatis dibuat di
`plugins/economy_bridge/config.yml` saat plugin pertama kali di-enable,
bersamaan dengan `sync_state.json` yang menyimpan baseline terakhir tiap
player (lihat "Aturan Resolusi Konflik" di atas). Kamu umumnya tidak perlu
mengubah `sync_state.json` secara manual; menghapusnya hanya me-reset
baseline semua player, sehingga rekonsiliasi berikutnya untuk masing-masing
mereka jatuh ke "saldo tertinggi menang" satu kali.

## Riwayat Perbaikan

Beberapa hal berikut sebelumnya menyebabkan plugin gagal di-load atau salah
secara logika, dan sudah diperbaiki di kode saat ini — dicatat di sini
sebagai referensi kalau kamu meng-copy pola kodenya ke plugin lain:

1. **`permissions` harus `dict`, bukan `list`** — `plugin_loader.py`
   memanggil `.items()` pada atribut ini, sama seperti `commands`.
2. **Jangan pakai `from __future__ import annotations` di file yang berisi
   `@event_handler`** — baris itu membuat semua type hint (termasuk tipe
   event) menjadi string biasa saat runtime, sehingga Endstone gagal
   mengenali `PlayerJoinEvent` lewat introspeksi signature.
3. **Sintaks enum di `usages` command**: `(value1|value2)<nama: TypeName>`
   — daftar pilihan di **depan** dalam kurung `()`, baru diikuti nama
   parameter & tipe di dalam `<...>` atau `[...]` (opsional).
4. **"Saldo tertinggi menang" secara naif merusak belanja** — versi awal
   sinkronisasi dua arah membandingkan Coin vs Money langsung tanpa
   mengingat state sebelumnya, sehingga diam-diam membatalkan setiap
   pembelian/pengurangan di salah satu sisi dalam satu siklus sync.
   Diperbaiki dengan menambahkan `SyncStateStore` (baseline tersimpan per
   player) agar perubahan asli — termasuk pengurangan — bisa dibedakan
   dari konflik asli.
5. **"Tertinggi menang" sebagai tie-break konflik juga membuang perubahan
   di salah satu sisi** — meski baseline sudah ada, memilih saldo yang
   lebih besar saat kedua sisi benar-benar berubah tetap menghapus
   perubahan yang lebih kecil (mis. pembelian terhapus oleh reward yang
   terjadi bersamaan). Diganti dengan penggabungan aditif kedua selisih
   (`baseline + Δcoin + Δmoney`, di-clamp minimal 0), yang menjaga efek
   kedua transaksi alih-alih membuang salah satunya.

## Bagian yang WAJIB Diverifikasi Ulang di Server Kamu

Karena API plugin pihak ketiga bisa berubah antar versi, cek kembali
(ditandai `# TODO` di kode):

1. `JWEconomyProvider.PLUGIN_NAME` — nama JWEconomy sebagaimana terdaftar
   di `plugin_manager`.
2. `JWEconomyProvider`: cara mendapatkan API (`get_api()`) dan signature
   `get_balance(uuid, currency)`.
3. `UMoneyProvider.PLUGIN_NAME` — nama uMoney sebagaimana terdaftar di
   `plugin_manager`.
4. `UMoneyProvider`: nama method `api_get_player_money` /
   `api_reset_player_money`.
