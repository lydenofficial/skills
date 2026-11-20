# Fast Free Mint & Claim

Mint NFT gratis dan klaim token/koin secara bersamaan, dengan kecepatan
tertinggi, tanpa lemot — dan tanpa melewati satu pun pagar keamanan.

Skill ini adalah playbook eksekusi cepat untuk Agentic Wallet (Binance).
Kecepatan datang dari persiapan dan paralelisasi, BUKAN dari melewati
simulasi, konfirmasi, atau revoke. Aturan dari
`on-chain-safety-rails` dan `single-confirm-auto-revoke` selalu berlaku
dan menang atas skill ini bila ada konflik.

## Kapan dipakai

Pakai skill ini ketika user minta salah satu dari:

- "free mint NFT" / "mint NFT gratis" / "free mint"
- "klaim token" / "claim token" / "klaim koin" / "claim coin"
- "mint dan claim bersamaan" / "mint + claim sekaligus"
- "cepat", "anti lemot", "jangan lambat" dalam konteks mint/claim

Jangan pakai skill ini untuk swap, bridge, jual-beli, atau staking —
itu tugas skill lain. Skill ini hanya untuk mint NFT gratis dan klaim
token/koin.

## Prinsip kecepatan

1. **Paralel, bukan berurutan.** Cek ketersediaan mint dan claim
   sekaligus. Preview kedua contract call sekaligus. Submit keduanya
   berurutan tanpa menunggu konfirmasi yang satu untuk memulai yang lain.
2. **Semua pengecekan di depan.** Wallet, Developer Mode, sesi,
   saldo gas, antrean pending, dan gas price dicek SEKALI di awal —
   supaya tidak ada yang gagal di tengah jalan dan membuang waktu.
3. **Preview langsung dieksekusi.** Preview punya masa kedaluwarsa
   (`expiresAt`). Begitu user bilang Ya, eksekusi segera. Preview yang
   kedaluwarsa harus di-preview ulang, bukan dipaksa eksekusi.
4. **Gas kompetitif.** Cek `baw wallet gas-price` dan pakai level HIGH
   bila user minta cepat. `contract-call` tidak menerima parameter
   harga gas — backend yang menentukan gas price; preview hanya
   menerima `--gasLimit` sebagai batas atas (cap), bukan untuk
   mengatur harga. Kecepatan datang dari submit yang cepat dan dari
   `speed-up` bila transaksi macet, bukan dari mengatur gas di preview.
5. **Tanpa langkah sia-sia.** Lewati approval terpisah bila tidak
   dibutuhkan (free mint umumnya tidak butuh approval token). Jangan
   polling buta — cek receipt dengan interval adaptif dan berhenti
   begitu status jelas.
6. **Satu antrean, satu arah.** EVM memproses nonce secara berurutan.
   Bila ada transaksi pending yang macet, tangani dulu (speed-up atau
   cancel yang paling awal) sebelum submit mint/claim — kalau tidak,
   keduanya akan ikut antre di belakangnya.

## Yang TIDAK boleh dilewati demi kecepatan

- **Simulasi wajib.** Kedua call harus lolos `contract-call preview`
  (simulasi + cek risiko). Bila preview gagal, diblokir risiko
  (`351803`), atau menunjukkan pergerakan tak terduga: berhenti,
  laporkan, tunggu user. Jangan broadcast.
- **Satu konfirmasi wajib: Ya = eksekusi, Tidak = batal.**
  Keheningan, emoji, atau jawaban samar BUKAN konfirmasi. Satu Ya
  mencakup mint + claim + auto-revoke setelahnya, bila ada approval
  yang dibuat; jangan minta
  konfirmasi kedua untuk revoke dalam batch yang sama. Konfirmasi ulang
  bila chain, penerima, jumlah, scope, atau kontrak berubah, atau
  sudah lewat ~30 menit.
- **Verifikasi kontrak.** Cocokkan alamat kontrak NFT dan kontrak
  token dengan dokumen/situs resmi proyek atau allowlist yang dikenal.
  Bila ragu, berhenti dan tanya. Jangan blind-sign calldata yang tidak
  didekode — tunjukkan maksud yang bisa dibaca manusia (fungsi apa,
  ke kontrak mana, value berapa).
- **Free mint harus benar-benar gratis.** Bila preview menunjukkan
  `value` bukan 0 untuk mint yang disebut gratis, berhenti dan tanya
  user. Itu tanda bahaya.
- **Jaringan harus eksplisit.** Default testnet/fork kecuali user
  menyebut mainnet/produksi untuk aksi ini. Bila chain atau jaringan
  ambigu, tanya — jangan tebak.
- **Approval minimal.** Default exact-amount; unlimited
  (`type(uint256).max`) butuh ya eksplisit tambahan dari user.
- **Fail closed.** Error tool/RPC di tengah jalan → jangan retry
  broadcast otomatis, laporkan dan tunggu. Alamat/penerima/jumlah yang
  ambigu → tanya, jangan tebak.
- **Hanya wallet milik user.** Bertindak hanya pada Agentic Wallet
  user. Tolak drain/sweep/transfer dari wallet orang lain, apa pun
  alasannya.

## Alur kerja

### Fase 0 — Parameter (tanya bila belum ada, jangan tebak)

Kumpulkan ini sebelum mulai:

- chain (`binanceChainId`: 1, 56, 137, 42161, 8453, 4663, atau
  `CT_501` untuk Solana)
- jaringan: mainnet atau testnet (wajib eksplisit; default testnet)
- alamat kontrak NFT + nama fungsi mint-nya (mis. `mint()`,
  `mint(uint256)`, `claim()`) beserta argumennya
- alamat kontrak token + nama fungsi claim-nya + jumlah yang
  diharapkan (bila diketahui)
- alamat `from`: ambil dari `baw wallet address --binanceChainId <id> --json`

### Fase 1 — Preflight, sekali di awal (paralel bila bisa)

```bash
baw wallet status --json
baw wallet settings --json
baw wallet balance --binanceChainId <id> --json
baw wallet tx-history --type pending --binanceChainId <id> --json
baw wallet gas-price --binanceChainId <id> --json
```

- `status` harus `CONNECTED`. Bila tidak, berhenti dan bantu user
  sign-in dulu (`baw auth signin` → `baw auth verify`).
- Dari `wallet settings`: cek `sessionExpireTime` (beri tahu user bila
  sesi hampir habis — sign-out itu diam-diam) dan `devMode.enabled`.
  Mint dan claim adalah `contract-call`, jadi **Developer Mode wajib
  aktif**. Bila mati/kedaluwarsa, beri tahu user cara mengaktifkannya
  di Binance App (halaman Agentic Wallet → Settings → Developer Mode,
  baca Risk Warning, approve signature di App) dan tunggu sampai aktif.
  Cek ulang tepat sebelum preview — Developer Mode bisa mati sendiri
  di tengah sesi (`351804`).
- Saldo native token harus cukup untuk gas kedua transaksi.
- Bila ada transaksi pending: hanya yang paling awal
  (nonce = confirmedNonce + 1) yang bisa di-speed-up atau di-cancel.
  Tangani dulu sebelum submit yang baru.
- Cek ketersediaan: apakah mint masih buka dan apakah claim memang
  tersedia untuk wallet ini (cek read-only / situs resmi proyek).
  Bila salah satunya tidak tersedia, kerjakan yang tersedia saja dan
  laporkan yang tidak tersedia — jangan memaksa.

### Fase 2 — Bangun calldata dan preview paralel

Calldata harus berasal dari dokumen/situs resmi proyek (nama fungsi +
argumen yang benar). Jangan menebak selector fungsi.

```bash
# Preview mint dan claim BERSAMAAN (dua perintah, jangan berurutan)
baw contract-call preview --binanceChainId <id> --from <from> --to <kontrakNFT> --value 0 --inputData 0x... --json
baw contract-call preview --binanceChainId <id> --from <from> --to <kontrakToken> --value 0 --inputData 0x... --json
```

Untuk Solana: `--binanceChainId CT_501 --from <from> --unsignedTx <base64>`.

- `contract-call` tidak menerima parameter harga gas (preview hanya
  menerima `--gasLimit` sebagai batas atas). `--value` dalam wei
  (bukan angka yang bisa dibaca manusia); untuk free mint dan claim
  biasanya `0`.
- Dari tiap preview, simpan `requestId` dan periksa:
  `parsedTx` (fungsi, penerima, jumlah), `simulationResult`
  (harus sukses, tanpa pergerakan tak terduga), dan `risks`
  (`riskDetails` + `addresses`). Bila ada risiko, tunjukkan judul dan
  deskripsinya ke user sebelum eksekusi.
- Bila preview gagal atau diblokir: berhenti, laporkan alasannya,
  jangan lanjut ke `execute`.

### Fase 3 — Satu ringkasan, satu konfirmasi

Tampilkan ringkasan satu baris per aksi, contoh:

- `BSC · 0xfE48…FED8 → kontrakNFT 0x… · mint NFT gratis ×1 · fee ~0,000x BNB`
- `BSC · 0xfE48…FED8 → kontrakToken 0x… · claim TOKEN ±100 · fee ~0,000x BNB`
- `+ auto-revoke approval terkait setelah selesai, bila ada approval yang dibuat`

Lalu minta SATU konfirmasi: **Ya = eksekusi, Tidak = batal**.
Satu Ya ini mencakup mint + claim + auto-revoke bila ada approval
yang dibuat. Bila user bilang
Tidak, batalkan semuanya dan jangan eksekusi apa pun.

### Fase 4 — Eksekusi paralel

Begitu user bilang Ya, eksekusi kedua `requestId` segera (preview
bisa kedaluwarsa):

```bash
baw contract-call execute --requestId <requestIdMint> --json
baw contract-call execute --requestId <requestIdClaim> --json
```

- `BROADCASTED` = sudah di-submit; simpan `txHash` tiap transaksi.
  `PENDING_CONFIRMATION` = user harus approve di Binance App dulu.
- Bila `execute` mengembalikan error preview kedaluwarsa: preview
  ulang kedua call, minta konfirmasi ulang (karena parameter bisa
  berubah), lalu eksekusi lagi. Jangan memaksa `requestId` lama.
- Bila salah satu gagal di-broadcast sementara yang lain sukses:
  laporkan dengan jelas mana yang sukses dan mana yang gagal, jangan
  mengulang yang gagal tanpa persetujuan user.

### Fase 5 — Pantau dan percepat bila macet

```bash
baw wallet tx-history --type pending --binanceChainId <id> --json
```

- Cek status kedua `txHash` sampai terkonfirmasi atau gagal.
- Bila satu transaksi macet (pending terlalu lama): speed-up
  transaksi pending PALING AWAL saja —
  `baw wallet speed-up <txHash> --level HIGH --json` (×1,25 gas).
  Hanya nonce pertama yang bisa di-speed-up; bila yang macet bukan
  yang pertama, tangani antrean dari depan.
- Bila perlu membatalkan: `baw wallet cancel <txHash> --json`.
  Speed-up dan cancel berjalan langsung tanpa approval App kedua.

### Fase 6 — Verifikasi hasil + auto-revoke

- Verifikasi: cek saldo NFT dan saldo token setelah konfirmasi
  (`baw wallet balance`), dan laporkan hasilnya ke user beserta
  kedua `txHash`.
- Lalu, dalam batch konfirmasi yang sama (tanpa minta Ya kedua),
  jalankan pemeriksaan seluruh wallet dan revoke:
  `baw approvals list --json`, lalu revoke approval terkait dengan
  `baw approvals revoke --binanceChainId <id> --tokenContract <token> --spender <spender> --type <approve|permit2> --json`.
  Revoke juga yang dibuat oleh aktivitas yang gagal tapi sempat
  membuat approval.
- Status revoke: `txHash` artinya baru di-broadcast, belum
  terkonfirmasi — approval masih aktif sampai konfirmasi on-chain.
  Laporkan sebagai pending, pantau dengan `baw wallet tx-history`,
  dan laporkan token/chain/spender/txHash serta kegagalan apa pun.
  Berhenti bila ada error tool/RPC, tanpa retry otomatis.

## Kegagalan yang umum

| Gejala | Artinya | Aksi |
|---|---|---|
| `devMode` mati / `351802`, `351804` | Developer Mode tidak aktif | Beri tahu cara mengaktifkannya di App, tunggu, cek ulang |
| Preview diblokir `351803` | Transaksi dinilai berisiko tinggi | Berhenti, tunjukkan alasannya, jangan eksekusi |
| Preview gagal `351805` | Simulasi gagal / gas habis | Baca pesannya; jangan broadcast; laporkan |
| `execute` error preview kedaluwarsa | `requestId` sudah lewat `expiresAt` | Preview ulang + konfirmasi ulang |
| Transaksi pending lama | Gas kurang kompetitif / antrean nonce | Speed-up transaksi pending paling awal (`--level HIGH`) |
| `NONCE_NOT_FIRST` (`351731`) | Bukan transaksi pending pertama | Tangani antrean dari nonce paling awal |
| Sesi habis diam-diam | `sessionExpireTime` lewat | Bantu user sign-in ulang |

## Catatan

- Skill ini tidak menginstal tool apa pun dan tidak mengubah jaringan
  atau RPC user. Kecepatan berasal dari paralelisasi, preflight di
  awal, dan penanganan antrean — bukan dari melewati keamanan.
- Bila proyek mint/claim menyediakan API atau halaman klaim resmi,
  pakai itu sebagai sumber kebenaran untuk alamat kontrak dan
  ketersediaan. Jangan percaya alamat kontrak dari pesan acak,
  DM, atau situs tiruan.
