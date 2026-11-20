# Total Cost Disclosure — Biaya Total Transparan

Untuk AKTIVITAS APAPUN yang melibatkan biaya — terutama semua aktivitas
wallet/on-chain (mint, claim, bridge, swap, buy-sell, listing, approve,
transfer, staking, dan lain-lain) — selalu tampilkan BIAYA
KESELURUHAN, yang terlihat MAUPUN yang tersembunyi, dalam tiga mata
uang sekaligus: kripto, Rupiah (IDR), dan Dolar AS (USD). Tanpa
kecuali. Biaya revoke otomatis (bila ada) dihitung sebagai biaya
tersembunyi, lihat bagian di bawah.

Aturan ini mengikat semua ringkasan konfirmasi. Konfirmasi tidak
lengkap tanpa biaya total. Aturan ini berlaku bersama
`on-chain-safety-rails`, `single-confirm-auto-revoke`, dan
`fast-free-mint-claim`, dan tidak boleh dilewati demi kecepatan.

## Kapan dipakai

- Setiap kali menyiapkan, merangkum, atau meminta konfirmasi untuk
  aktivitas berbayar apa pun.
- Setiap kali melaporkan hasil aktivitas: tampilkan biaya AKTUAL yang
  terjadi, dan sebutkan selisihnya bila jauh dari estimasi.
- Untuk aktivitas non-kripto yang berbayar (mis. booking, belanja),
  tampilkan juga total lengkap termasuk pajak/fee tersembunyi, dalam
  mata uang pembayaran + padanan IDR dan USD bila relevan.

## Yang harus ditampilkan

Untuk setiap aktivitas, tampilkan tiga bagian ini:

1. **Biaya terlihat (visible):** gas transaksi utama, harga mint/beli,
   fee protokol/layanan/platform, fee bridge, dan semua biaya yang
   tertulis di muka.
2. **Biaya tersembunyi (hidden):** gas approval (approve/permit) bila
   dibutuhkan, gas auto-revoke sesudahnya, slippage / price impact
   pada swap, spread, fee konversi, dan biaya lain yang muncul di
   hasil preview (`simulationResult.balanceChanges`,
   `allowanceChanges`). Biaya revoke otomatis dari
   `single-confirm-auto-revoke` masuk hitungan bila ada revoke —
   yaitu untuk aktivitas yang membuat approval (bridge, swap,
   buy/sell, paid mint, listing). Revoke itu transaksi on-chain dan
   ada biayanya, jangan pernah menyembunyikannya bila ada.
3. **Total keseluruhan** = jumlah semua di atas, dalam tiga mata uang
   sekaligus. Format baku:

   `Total: ±0,0004 BNB ≈ Rp3.300 ≈ $0,20`

   (Angka di atas hanya contoh format — selalu hitung dari data nyata
   saat itu.)

Contoh ringkasan konfirmasi yang benar (mint gratis):

- `BSC · 0xfE48…FED8 → kontrakNFT 0x… · mint NFT gratis ×1`
- `Biaya: gas ±0,0004 BNB (≈ Rp3.300 ≈ $0,20)`
- `Total: ±0,0004 BNB ≈ Rp3.300 ≈ $0,20`

Untuk aktivitas yang membuat approval (bridge, swap, buy/sell, paid
mint, listing), tambahkan baris revoke, mis. `+ revoke ±0,0002 BNB
(≈ Rp1.650 ≈ $0,10)`, dan masukkan ke total.

## Cara mendapatkan angka — jangan menebak

- **Biaya gas dan fee:** dari preview/quote perintah yang bersangkutan
  (`contract-call preview` → `simulationResult`,
  `gasTokenFromData`, `balanceChanges`; `baw wallet gas-price` untuk
  level gas; `defi preview` / quote swap untuk fee dan slippage).
  Jangan mengalikan harga gas × gas limit sendiri bila perintahnya
  tidak memberi total — pakai total dari preview/quote.
- **Harga kripto dalam USD:** dari `tokenInfos.price` di hasil
  preview, atau dari `baw wallet balance` bila menampilkan nilai fiat.
- **Kurs USD → IDR:** ambil dari sumber live saat itu (pencarian
  harga / API kurs). Selalu sebutkan kurs dan waktu yang dipakai,
  mis. `kurs 1 USD = Rp16.500 (18 Nov 2026, 15:50 WITA)`.
- **Bila harga atau kurs live tidak bisa didapat:** katakan dengan
  jelas bahwa padanan fiatnya belum terverifikasi — JANGAN mengarang
  angka. Tampilkan dulu jumlah kriptonya, ambil kursnya, baru minta
  konfirmasi Ya.
- **Estimasi vs aktual:** labeli estimasi sebagai estimasi (±). Biaya
  aktual bisa berubah sampai transaksi di-broadcast. Bila biaya aktual
  sesudah eksekusi berbeda jauh dari estimasi, laporkan selisihnya.

## Aturan

- Tidak ada aktivitas berbayar yang diminta konfirmasinya tanpa biaya
  total dalam tiga mata uang. Bila biayanya belum bisa dihitung,
  hitung dulu — jangan minta konfirmasi buta.
- Bila suatu aktivitas ternyata gratis total (0 gas pun tidak ada,
  mis. klaim off-chain), katakan eksplisit `Total: 0 (gratis)` —
  jangan diam saja.
- Bila ada biaya yang tidak bisa diketahui di muka (mis. gas revoke
  yang tergantung kondisi jaringan nanti), sebutkan sebagai rentang
  atau perkiraan dengan batas atas, dan tandai sebagai belum pasti.
- Format angka Indonesia: koma desimal, titik ribuan —
  `Rp5.000`, `0,0006 BNB`, `$0,30`.
