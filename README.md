# ZXOXO Bot — LYID

Bot farming untuk **ZXOXO** (`zxoxo.xyz`), test network mining bertema LayerZero.
Bot menjalankan pass harian per akun: **claim ZXO → beli rig → overclock → jump (kalau layak)**,
plus memantau skor **access signal** (whitelist/access packet).

Ditulis Python stdlib — **tanpa dependency**, tanpa `npm install`, tanpa browser, tanpa private key.

---

## Apa itu ZXOXO (hasil riset 13 Nov 2026)

| Item | Nilai |
|---|---|
| Situs | https://zxoxo.xyz |
| Akun X | [@zxoxocash](https://x.com/zxoxocash) — akun **baru**, 1 post, bio "Something is moving between chains · Coming soon on @LayerZero_Core" |
| Funding / investor | **tidak diumumkan** |
| Token | belum ada. Situs sendiri menulis *"Test ZXO has no value and nothing here touches a chain"* |
| Chain | tidak ada transaksi on-chain. "Chain" (ARB/BASE/OP/BNB/AVAX) hanya zona di dalam game |
| Wallet | hanya **alamat publik** untuk seal packet — tidak ada signature, tidak ada connect, tidak ada approve |
| Program | "Access / Phase 0" — 1 packet per akun X, peringkat berdasarkan **signal** |
| Papan publik | hanya 20 besar ditampilkan; batas masuk 20 besar ≈ 5.600 signal, teratas ≈ 15.100 |
| Peserta | ~500 miner, ~500+ pelamar access |
| Modal | **Rp 0** |

**Kesimpulan jujur:** ini proyek pre-launch anonim tanpa funding, tanpa token, tanpa dokumen.
Skornya rendah untuk kriteria LYID (funding 0, tim anonim, sosial 1 post), tapi biaya farming
benar-benar nol — tidak ada dana, tidak ada gas, tidak ada seed phrase yang disentuh. Anggap ini
**tiket lotre gratis**, bukan sumber penghasilan. Bot dibuat karena otomasinya murah, bukan karena
reward-nya pasti.

### Rumus signal (dibaca langsung dari bundle aplikasi)

```
signal = age    + task  + mining        + post   + relays
        250/thn   500     1 per 10 ZXO    1000     500 per referral
                          (maks 5.000)             (maks 20 → 10.000)
```

Artinya: **relay/referral adalah sumber signal terbesar** (sampai 10.000), bukan mining.
Mining mentok di 5.000 setelah 50.000 ZXO. Bot ini mengejar jalur mining karena itu satu-satunya
yang bisa diotomasi; jalur relay butuh akun X nyata yang join lewat link card kita.

---

## Fitur

- **Multi-akun** (cookie `zxoxo_member` per akun, tidak ada private key sama sekali)
- **Humanisasi per wallet** — persona stabil dari hash handle: ambang claim, tempo, agresivitas beli/OC,
  margin jump, reserve saldo, peluang libur (lihat tabel `--personas`)
- **Jeda log-normal** antar aksi (bukan `random.uniform` datar), jeda panjang sesekali, stagger antar akun,
  jitter di awal run supaya cron tidak menembak tepat 08:00:00
- **Libur acak** — tiap wallet punya peluang 5–18% tidak jalan di hari tertentu
- **Strategi hemat** — beli rig hanya kalau saldo lewat harga + reserve; overclock hanya kalau wajar;
  jump hanya untuk misi atau kalau gain-nya melebihi ongkos transit 10 menit
- **Mode dry-run** (`--dry-run`) untuk melihat keputusan tanpa mengubah state
- **Laporan** ke `data/last-run.json` + `data/last-run.txt` (bahan screenshot/README)
- **Launcher .bat** untuk yang tidak nyaman CLI

## Kebutuhan

| Item | Wajib? | Biaya |
|---|---|---|
| Python 3.9+ | Ya | Gratis |
| `pip install` apa pun | **Tidak** | — |
| Wallet / private key | **Tidak** | — |
| Modal / gas | **Tidak** | Rp 0 |
| Akun X (umur makin tua makin besar signal) | Ya (sudah ada) | Gratis |
| Proxy | Opsional (lihat bagian Humanisasi) | — |

## Setup

1. Buka https://zxoxo.xyz → **Connect X** → login dengan akun X yang mau dipakai.
2. Tekan `F12` → tab **Application** → **Cookies** → `https://zxoxo.xyz` → cari **`zxoxo_member`**.
3. Copy value-nya (bentuknya `1797xxx.1728xxx.xxxx.yyyy`).
4. Buka `config.json`, isi per akun:

```json
{
  "label": "akun1",
  "handle": "handleX",
  "address": "0xalamatEVMkamu",
  "cookie": "zxoxo_member=PASTE_DISINI"
}
```

5. Jalankan: klik `CEK-POIN.bat` (baca saja) atau `JALANKAN.bat` → menu 3.

Cookie ini setara password akun. `config.json` sudah masuk `.gitignore` — jangan pernah di-commit.

## Pemakaian

```bash
python main.py --status            # cek saldo, rig, signal, status access semua akun
python main.py --access            # detail whitelist: rank, task, post, relay, rincian signal
python main.py --personas          # tabel persona humanisasi + contoh jeda
python main.py --daily             # pass harian (LIVE)
python main.py --daily --dry-run   # simulasi, tidak mengubah apa pun
python main.py --daily --limit 1   # satu akun saja
python main.py --daily --no-jitter # tanpa jeda awal/stagger (untuk tes)
python main.py --daily --max-actions 3
```

Menu `JALANKAN.bat`:

```
[1] Cek status semua akun            (baca saja, aman)
[2] Detail akses / whitelist         (baca saja, aman)
[3] Jalankan pass harian             (LIVE - claim, beli rig, overclock, jump)
[4] Simulasi pass harian             (dry-run, tidak mengubah apa pun)
[5] Tabel persona humanisasi         (baca saja, aman)
[0] Keluar
```

## Mekanik yang dipakai bot

| Aksi | Endpoint | Efek |
|---|---|---|
| `claim` | `POST /api/mine/act {type:"claim"}` | ZXO hasil mining masuk ke saldo |
| `buy` | `{type:"buy", zone}` | beli rig baru. Harga per jumlah rig: **0, 800, 2.400, 6.000, 12.000, 24.000** ZXO (maks 6 rig) |
| `overclock` | `{type:"overclock", rig}` | +15% hash, bakar **300 × 2^level** ZXO (maks level 5) |
| `jump` | `{type:"jump", rig, zone}` | pindah chain: 10 menit transit + 2 jam cooldown |
| `start` | `{type:"start", zone}` | rig gratis 100 TH/s (sekali) |

Yield tiap chain = `2000 × hash / (totalHashZona + hash)` ZXO per jam. Karena semua chain hampir sama
(13,4–14,5), pindah chain jarang sepadan — bot hanya jump untuk misi atau kalau selisihnya > ongkos transit.

### Misi (masing-masing +150 ZXO)

`start` · `first claim` · `chase the yield` (jump) · `overclock` · `expand` (2 rig) · `omnichain` (3 chain)

Bitmask misi dibaca dari `me.missions`, jadi bot tidak pernah mengulang misi yang sudah selesai.

## Humanisasi (dan apa yang TIDAK diperbaiki)

Yang dirandomisasi per wallet, stabil sepanjang umur akun:

| Knob | Rentang | Efek |
|---|---|---|
| `claimAt` | 0,4–24 ZXO | ambang "baru buka app" sebelum claim |
| `tempo` | 0,60–1,95 | pengali semua jeda wallet itu |
| `jumpMargin` | 0,12–0,97 | seberapa besar selisih yield sebelum mau pindah |
| `buyEager` / `ocEager` | 0,35–1,0 / 0,25–0,95 | agresivitas belanja |
| `reserve` | 35–295 ZXO | saldo yang tidak dihabiskan, biar tidak semua wallet berakhir di 0,00 |
| `restDay` | 5–18% | peluang wallet tidak jalan hari itu |

Plus: jeda **log-normal** (bukan range datar), satu jeda panjang 20–90 detik sesekali, urutan misi
di-shuffle per wallet, dan stagger acak 25–210 detik antar akun.

**Yang tetap terbuka dan harus kamu tahu:**

- **Satu IP untuk semua akun.** Ini sinyal terkuat dan tidak diperbaiki oleh jitter apa pun.
  Kalau mau serius, proxy residensial per akun (kolom `proxy` di `config.json` sudah disiapkan).
- **Kelima packet access sudah terkirim dari IP yang sama** dalam hitungan menit, ke post yang sama.
  Itu sudah terjadi sebelum bot ini ada dan tidak bisa dibatalkan.
- Umur akun X dan riwayat handle adalah apa adanya — persona mengubah tekstur perilaku, bukan topologi.
- Bot tidak pernah mengklaim "tidak terdeteksi". Yang berubah: bentuk traffic. Yang tidak: IP, funding
  graph (tidak ada, karena tidak ada transaksi), dan waktu submit.

## Struktur file

```
zxoxo-bot/
├── main.py                 # CLI + menu
├── src/
│   ├── api.py              # klien HTTP + konstanta game (harga rig, biaya OC, misi)
│   ├── humanize.py         # persona, log-normal delay, aturan jump/claim/beli
│   └── runner.py           # pass harian per akun + laporan
├── config.json             # cookie akun (GITIGNORED)
├── config.json.example     # template
├── data/                   # last-run.json, last-run.txt
├── JALANKAN.bat            # menu
├── CEK-POIN.bat            # cek status cepat
├── CEK-ACCESS.bat          # cek whitelist cepat
├── PANDUAN-MANUAL.md       # arti menu + arti pesan error
├── README.md
└── CHECKPOINT.md
```

## Troubleshooting

| Pesan | Arti | Solusi |
|---|---|---|
| `signed_out` / `cookie expired` | cookie `zxoxo_member` sudah tidak valid | login X lagi di browser, ambil cookie baru, update `config.json` |
| `rate_limited` | server menolak karena terlalu cepat | bot menunggu sendiri 45–75 detik lalu mencoba sekali lagi |
| `not_enough_zxo` | saldo kurang untuk beli/overclock | normal, bot lanjut saja |
| `nothing_to_claim` | hasil mining masih di bawah 0,01 ZXO | normal |
| `cooling_down` / `in_transit` | rig baru jump, masih 10 menit transit / 2 jam cooldown | normal, bot skip sampai waktunya |
| `max_rigs` | sudah 6 rig | normal |
| `forbidden` | header Origin ditolak | pastikan request lewat `src/api.py` (Origin/Referer sudah diset) |
| `http_5xx` | server sedang gangguan | bot retry sekali; kalau tetap gagal, jalankan lagi nanti |

## Status verifikasi

| Bagian | Status | Bukti |
|---|---|---|
| Auth cookie | **Verified** | `GET /api/mine/state` → `signedIn: true` untuk 5 akun |
| `claim` | **Verified** | live: acc03 bal 300,14 → 459,30 |
| `jump` | **Verified** | rig 440 optimism → arbitrum; setelah `arriveAt` lewat, `toZone: null`, `zone: arbitrum`, `rate` 0 → **16,1 ZXO/jam** |
| `overclock` | **Verified** | rig 440 ke level 1, -300 ZXO, misi Overclock selesai |
| Misi (+150 ZXO) | **Verified** | `mined` naik tepat 150 setelah misi Overclock |
| `buy` (rig baru) | **Belum** | saldo belum mencapai 800 ZXO di akun mana pun saat build |
| Humanisasi | **Verified (unit)** | tabel `--personas` + 5 draw jeda per wallet berbeda |
| Cron harian | lihat `CHECKPOINT.md` | — |

`buy` belum pernah tereksekusi karena belum ada akun yang punya 800 ZXO. Kodenya memakai ladder harga
yang sama dengan bundle, dan akan jalan sendiri begitu saldo cukup.

---

Dibuat oleh **LYID (Layer Airdrop ID)** · [@rmndkyl](https://github.com/rmndkyl)
Bot ini off-chain dan tidak memegang private key. Cookie akun = akses penuh, jaga `config.json`.
