# NFT Auto-Mint Bot

Bot auto-mint NFT modular untuk EVM (Ethereum, Base, Arbitrum, Optimism) dan Solana.
Mendukung deteksi fase mint otomatis, multi-wallet, retry dengan backoff, notifikasi
Telegram, dry-run mode, dan whitelist mint dengan merkle proof.

> ⚠️ **Disclaimer**: Bot ini adalah tooling otomasi transaksi on-chain yang secara
> teknis identik dengan mint manual lewat wallet (Metamask/Phantom), hanya
> dieksekusi lewat script. Pastikan penggunaan lo sesuai Terms of Service project
> NFT yang lo mint dan regulasi yang berlaku di wilayah lo. Selalu test dengan
> `dry_run: true` sebelum live, dan JANGAN pernah commit `.env` / private key ke
> repository manapun.

---

## 📁 Struktur Folder

```
nft_automint_bot/
├── main.py                        # Entry point utama
├── requirements.txt
├── .env.example                   # Template environment variables
├── config/
│   ├── config.example.yaml        # Template konfigurasi mint
│   ├── config_loader.py           # Loader & validator config
│   └── merkle_proofs.example.json # Template proof whitelist
├── core/
│   ├── chain_base.py              # Interface abstrak chain adapter
│   ├── evm_chain.py                # Implementasi EVM (web3.py)
│   └── solana_chain.py             # Implementasi Solana (solders)
├── bot/
│   ├── mint_bot.py                 # Orchestrator utama
│   ├── phase_detector.py           # Deteksi fase mint (auto + manual fallback)
│   └── merkle.py                   # Provider merkle proof (file/API)
├── utils/
│   ├── logger.py                   # Setup logging (console + file)
│   ├── telegram_notifier.py        # Notifikasi Telegram
│   ├── retry.py                    # Retry + backoff + klasifikasi error
│   └── abi_resolver.py             # Auto-detect ABI mint function (EVM)
├── scripts/
│   ├── check_balance.py            # Cek balance semua wallet
│   └── estimate_gas.py             # Estimasi gas sebelum mint
└── logs/                           # Auto-generated log files
```

---

## 🚀 Instalasi

```bash
# 1. Clone / masuk ke folder project
cd nft_automint_bot

# 2. Buat virtual environment (disarankan)
python3 -m venv venv
source venv/bin/activate      # Linux/Mac
# venv\Scripts\activate       # Windows

# 3. Install dependencies
pip install -r requirements.txt
```

---

## ⚙️ Konfigurasi

### 1. Environment Variables (`.env`)

```bash
cp .env.example .env
```

Isi `.env` dengan:
- Private key wallet (EVM dan/atau Solana, pisahkan multi-wallet dengan koma)
- RPC URL (Alchemy/Infura/QuickNode untuk EVM, Helius/QuickNode untuk Solana)
- Token & Chat ID bot Telegram
- (Opsional) Private RPC untuk anti-MEV (misal Flashbots Protect)

### 2. Config Mint (`config.yaml`)

```bash
cp config/config.example.yaml config/config.yaml
```

Isi field penting:

| Field | Keterangan |
|---|---|
| `chain` | `"evm"` atau `"solana"` |
| `evm.contract_address` | Alamat smart contract NFT |
| `evm.mint_function.name` / `args` | Nama & argumen fungsi mint (lihat ABI di block explorer). **Sangat disarankan diisi manual**, jangan andalkan auto-detect untuk mint kompetitif |
| `evm.price_per_nft_eth` | Harga per NFT |
| `execution.total_mint_count` | Total NFT yang mau di-mint |
| `execution.dry_run` | **WAJIB `true`** saat testing pertama kali |
| `whitelist.enabled` | `true` kalau mint whitelist pakai merkle proof |

### 3. Merkle Proof (kalau whitelist)

```bash
cp config/merkle_proofs.example.json config/merkle_proofs.json
```

Isi dengan mapping `wallet_address -> [proof_array]`. Biasanya project
mempublish file ini di GitHub/website resmi mereka, atau sediakan API endpoint
(set `whitelist.proof_source: "api"` di config).

---

## ▶️ Cara Pakai

### Cek balance semua wallet
```bash
python scripts/check_balance.py
```

### Estimasi gas sebelum mint
```bash
python scripts/estimate_gas.py --qty 1
```

### Dry-run (simulasi, TIDAK ada transaksi asli)
```bash
python main.py --dry-run
```
Cek log di terminal & `logs/mintbot_YYYYMMDD.log` — pastikan:
- Fase mint terdeteksi dengan benar
- Fungsi mint yang di-resolve sesuai ekspektasi
- Estimasi gas & value wajar
- Simulasi `eth_call` sukses (untuk EVM)

### Live mint (transaksi ASLI)
1. Set `execution.dry_run: false` di `config.yaml`
2. Jalankan:
```bash
python main.py
```
3. Bot akan minta konfirmasi ketik `YES` sebelum mulai (safety net tambahan).

---

## 🔁 Alur Kerja Bot

1. **Load config & wallet** dari `.env` + `config.yaml`
2. **Tunggu fase mint** — polling `detect_mint_phase()` tiap `poll_interval_seconds`,
   fallback ke `manual_schedule` kalau contract tidak expose status function
3. **Untuk tiap wallet**:
   - Cek balance
   - Ambil merkle proof (kalau whitelist mint)
   - Approve token ERC20 kalau `approve_token.enabled: true`
   - Build & kirim transaksi mint (via private RPC kalau anti-MEV aktif)
   - **Retry otomatis** kalau error retryable (gas too high, network congestion,
     nonce error, rpc timeout) dengan exponential backoff
   - Kirim notifikasi Telegram (sukses/gagal)
   - Random delay sebelum wallet berikutnya
4. Ulangi sampai `total_mint_count` tercapai atau wallet habis

---

## 🛡️ Anti-MEV & Retry

- **Anti-MEV (EVM)**: kalau `EVM_PRIVATE_TX_RPC_URL` diisi di `.env` dan
  `evm.anti_mev.enabled: true`, transaksi mint dikirim lewat private RPC
  (contoh: Flashbots Protect, MEV Blocker) supaya tidak nongol di public
  mempool dan rawan di-frontrun/sandwich.
- **Retry**: error diklasifikasikan otomatis (`utils/retry.py`) jadi:
  - **Retryable**: gas too high, network congestion, nonce error, rpc timeout
    → di-retry dengan exponential backoff + jitter
  - **Fatal**: insufficient funds, sold out, invalid proof, revert generik
    → langsung stop, tidak buang-buang gas retry sia-sia

---

## 🧩 Menambah Chain Baru

1. Buat file baru di `core/`, misal `core/sui_chain.py`
2. Extend class `ChainAdapter` dari `core/chain_base.py`, implement semua
   method abstrak (`load_wallets`, `get_balance`, `detect_mint_phase`, `mint`, dst)
3. Register di `bot/mint_bot.py::build_chain_adapter()`

---

## ⚠️ Catatan Penting — Solana

Berbeda dengan EVM yang punya ABI universal, program mint NFT di Solana
(Candy Machine v3, custom Anchor program) punya instruction encoding yang
spesifik per program. Adapter `core/solana_chain.py` menyediakan skeleton
lengkap (koneksi, wallet, balance, priority fee, retry) tapi method
`_build_candy_machine_v3_instruction()` perlu lo isi sesuai IDL program
target (pakai `anchorpy` + IDL resmi, atau reverse-engineer dari transaksi
mint sukses di Solscan). Ini didokumentasikan jelas di dalam kode.

---

## 🐛 Troubleshooting

| Masalah | Kemungkinan Penyebab |
|---|---|
| `Auto-detect gagal menemukan mint function` | Set `mint_function.name` & `args` manual di config, cek ABI di block explorer |
| `insufficient funds` (fatal, tidak retry) | Balance wallet kurang, cek `scripts/check_balance.py` |
| Semua attempt retry habis dengan `gas_too_high` | Naikkan `priority_fee_bump_gwei` atau cek kondisi network |
| Telegram tidak kirim notifikasi | Cek `TELEGRAM_BOT_TOKEN` & `TELEGRAM_CHAT_ID` di `.env`, pastikan bot sudah di-`/start` di chat tsb |
| Solana mint `NotImplementedError` | Isi `_build_candy_machine_v3_instruction()` sesuai IDL program target |

---

## 📜 Lisensi

Untuk penggunaan pribadi/riset. Modifikasi bebas sesuai kebutuhan project lo.
