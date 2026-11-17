# Server-Side Font Protection

Panduan untuk menampilkan preview font privat di web tanpa mengirim file font
asli (`.ttf`, `.otf`, atau format sumber lain) ke browser.

Skill ini menyediakan tiga pilihan:

1. **SVG path** — teks diubah menjadi outline SVG dengan `opentype.js`.
2. **PNG/WebP** — teks dirender menjadi gambar dengan `satori` dan `sharp`.
3. **Dynamic subset** — hanya glyph yang sedang dipakai dikirim sebagai WOFF2
   dengan `subset-font`.

> Tidak ada pendekatan yang menjadi DRM sempurna. Raster paling sulit
> diekstrak, SVG masih berisi outline, dan subset dapat dipanen melalui banyak
> request.

## Instalasi sebagai AI Agent Skill

Bagian ini memasang folder skill agar dapat ditemukan dan digunakan oleh AI
agent. Ini berbeda dari instalasi package JavaScript pada bagian
[Instalasi dependency](#instalasi-dependency).

### Opsi 1: Instalasi dari repository Git

Clone repository ke direktori skills agent:

```bash
mkdir -p ~/.agents/skills
git clone https://github.com/rendrabagasdev/server-font-protection.git \
  ~/.agents/skills/server-font-protection
```

Jika folder skill sudah ada, perbarui dengan:

```bash
cd ~/.agents/skills/server-font-protection
git pull --ff-only
```

### Opsi 2: Instalasi dari folder lokal

Jika folder ini sudah tersedia secara lokal:

```bash
mkdir -p ~/.agents/skills
cp -R /path/ke/server-font-protection ~/.agents/skills/server-font-protection
```

Struktur minimal yang wajib ada:

```text
~/.agents/skills/server-font-protection/
├── SKILL.md
├── README.md
└── references/
    ├── 01-svg-paths.md
    ├── 02-image-render.md
    ├── 03-font-subsetting.md
    └── 04-security-and-performance.md
```

### Verifikasi instalasi

```bash
test -f ~/.agents/skills/server-font-protection/SKILL.md \
  && echo "Skill terpasang"
```

Pastikan `SKILL.md` memiliki frontmatter dengan:

```yaml
name: server-font-protection
```

### Aktifkan skill pada AI agent

1. Restart atau reload sesi AI agent setelah folder skill ditambahkan.
2. Pastikan agent dapat membaca `SKILL.md` dan folder `references/`.
3. Minta agent melakukan tugas yang memicu skill, misalnya:

   > Buat endpoint preview font yang tidak mengirim file TTF ke browser.

4. Agent seharusnya memilih salah satu pola SVG path, raster PNG/WebP, atau
   dynamic font subset dan mengikuti batasan keamanan di skill.

Jika agent mendukung pemanggilan skill eksplisit, gunakan nama:

```text
server-font-protection
```

Jika skill belum terdeteksi, periksa kembali path instalasi, nama folder,
frontmatter `SKILL.md`, lalu restart agent. Jangan menaruh font privat di
folder skill; folder skill hanya berisi instruksi dan referensi.

## Kompatibilitas

### Runtime server

| Runtime | Status | Catatan |
|---|---|---|
| Node.js 18+ | Didukung | Runtime utama untuk semua pola. |
| Node.js 20/22/24 | Didukung | Disarankan untuk deployment baru; uji native dependency di target deployment. |
| Bun 1.x | Didukung dengan verifikasi | Instal dependency memakai `bun`; uji `sharp` dan WASM `subset-font` pada versi Bun serta OS target. |
| Deno | Bersyarat | Bisa melalui kompatibilitas npm/Node, tetapi `sharp` dan WASM harus diuji pada deployment aktual. |
| Cloudflare Workers / Vercel Edge / Deno Deploy Edge | Tidak didukung untuk semua pola | `sharp`, `opentype.js` berbasis Node, dan subsetter WASM memerlukan Node runtime. |
| Serverless berbasis Node | Didukung | Sertakan font privat, binary `sharp`, dan asset WASM dalam tracing/package. |
| Dedicated server, container, atau VM | Didukung | Pastikan binary native Sharp cocok dengan OS/arsitektur. |

### Framework dan adapter

Implementasi inti tidak bergantung pada framework. Endpoint cukup menerima
`Request` dan mengembalikan `Response`.

| Framework/platform | Status | Implementasi |
|---|---|---|
| Next.js App Router | Didukung langsung | Gunakan `app/api/**/route.ts` dan `export const runtime = "nodejs"`. |
| Next.js Pages Router | Didukung | Bungkus handler pada `NextApiRequest`/`NextApiResponse`. |
| Remix | Didukung | Gunakan `loader`/`action` pada Node adapter. |
| SvelteKit | Didukung | Gunakan `+server.ts` pada adapter Node. |
| Nuxt/Nitro | Didukung pada Node preset | Gunakan server route Nitro, bukan Edge preset. |
| Astro SSR | Didukung pada Node adapter | Jalankan endpoint pada server runtime Node. |
| Express | Didukung | Adapter `req`/`res` ke input teks dan output `res.type(...).send(...)`. |
| Fastify | Didukung | Route handler Fastify dengan response buffer/string. |
| Hono | Didukung pada Node/Bun | Gunakan adapter Node/Bun; jangan deploy ke Workers untuk pola ini. |
| Standalone Node HTTP | Didukung | Gunakan `node:http` dan panggil fungsi inti secara langsung. |
| Frontend statis saja | Tidak cukup | Tetap membutuhkan backend privat untuk membaca dan memproses font. |

Status “didukung dengan verifikasi” berarti API JavaScript bisa kompatibel,
tetapi dependency native atau WASM harus diuji pada runtime, OS, dan arsitektur
yang benar-benar dipakai.

### Format font

| Format input | SVG path (`opentype.js`) | Satori + Sharp | `subset-font` |
|---|---:|---:|---:|
| TTF | Didukung | Didukung | Didukung |
| OTF (CFF/TrueType) | Didukung untuk format yang didukung versi library | Didukung | Didukung |
| WOFF | Didukung | Didukung | Didukung bila decoder tersedia |
| WOFF2 | Tidak langsung; dekompresi dulu | Didukung dari `Buffer`/`ArrayBuffer` | Didukung sebagai input pada konfigurasi yang sesuai |
| Variable font | Terbatas; gunakan instance/format yang diuji | Default instance; atur axis sebelum render bila perlu | Didukung dengan `variationAxes` |

Untuk hasil yang paling konsisten, simpan sumber privat sebagai TTF atau OTF,
dan kirim hasil transformasi saja.

### Browser dan output

| Browser | SVG | PNG | WebP | WOFF2 + `FontFace` |
|---|---:|---:|---:|---:|
| Chrome/Edge modern | Didukung | Didukung | Didukung | Didukung |
| Firefox modern | Didukung | Didukung | Didukung | Didukung |
| Safari modern | Didukung | Didukung | Didukung | Didukung |
| Browser lama tanpa WebP | Didukung | Didukung | Gunakan fallback PNG | Gunakan fallback output atau browser support policy |
| JavaScript dinonaktifkan | `<img>` SVG/PNG/WebP tetap tampil | Didukung | Didukung | Tidak didukung |

PNG adalah pilihan paling kompatibel untuk gambar. WOFF2 cocok untuk browser
modern, tetapi tetap mengirim sebagian data font sehingga bukan proteksi penuh.

## Instalasi dependency

### Bun

```bash
bun add opentype.js satori sharp subset-font
```

### npm, pnpm, atau Yarn

```bash
npm install opentype.js satori sharp subset-font
# atau: pnpm add opentype.js satori sharp subset-font
# atau: yarn add opentype.js satori sharp subset-font
```

`sharp` dapat sudah terpasang sebagai dependency framework. Tetap pastikan
binary-nya tersedia pada target deployment.

## Struktur yang disarankan

```text
assets/
  fonts/                 # privat; jangan di bawah public/
    Inter-Display.otf
lib/
  fonts.ts               # allowlist font dan cache loader
app/api/font/
  svg/route.ts
  image/route.tsx
  subset/route.ts
```

Jangan menerima path file dari client. Gunakan allowlist `fontId -> path`,
batasi input teks, dan jangan mengekspos stack trace atau path font pada error.

## Memilih pola

| Kebutuhan | Pilihan |
|---|---|
| Preview real-time, tajam pada semua ukuran, boleh mewariskan warna CSS | SVG path |
| Font dan outline tidak boleh dikirim ke browser | PNG/WebP |
| Teks harus selectable/copyable dan kompromi proteksi dapat diterima | Dynamic subset |

Semua route harus berjalan pada Node runtime, memakai validasi input, rate
limit, cache yang sesuai sensitivitas teks, dan `X-Content-Type-Options:
nosniff`.

## Deployment checklist

- [ ] Font berada di folder privat, bukan `public/`.
- [ ] Tidak ada `@font-face` atau `next/font/local` yang menunjuk font asli.
- [ ] `fontId` memakai allowlist; tidak ada path bebas dari query.
- [ ] Teks, ukuran, warna, skala, dan format divalidasi serta di-clamp.
- [ ] Rate limit diterapkan pada endpoint publik.
- [ ] Route memakai Node runtime, bukan Edge runtime.
- [ ] Font, binary Sharp, dan WASM subsetter ikut package/tracing production.
- [ ] Cache publik hanya digunakan jika teks tidak sensitif dan key immutable.
- [ ] Production build sudah diuji pada OS/arsitektur target.
- [ ] Aksesibilitas tetap disediakan: `alt` untuk gambar atau teks fallback
      untuk preview yang tidak selectable.

## Referensi

- [SKILL.md](SKILL.md) — aturan operasional dan contoh ringkas.
- [SVG paths](references/01-svg-paths.md) — loader, path, viewBox, dan cache.
- [Image render](references/02-image-render.md) — Satori, Sharp, PNG/WebP,
  ukuran canvas, dan aksesibilitas.
- [Font subsetting](references/03-font-subsetting.md) — WOFF2, `FontFace`,
  cache, dan mitigasi harvesting glyph.
- [Security and performance](references/04-security-and-performance.md) —
  threat model, header, deployment, dan checklist produksi.

## Batasan keamanan

SVG path dapat diambil dari atribut `d` dan direkonstruksi. Subset dapat
dikumpulkan dengan request berulang. Raster menghilangkan outline font dari
response, tetapi teks tetap dapat dikenali dengan OCR. Karena itu gunakan
raster untuk font yang paling sensitif dan anggap semua output server sebagai
konten yang pada akhirnya dapat diamati oleh client.
