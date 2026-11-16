---
name: server-font-protection
version: 1.0.0
description: >-
  Implement server-side font protection for web previews: convert user-typed
  text into inline SVG vector paths with opentype.js, rasterize text to PNG or
  WebP with Satori + Sharp, and generate per-request subset fonts with
  subset-font. Use when the user wants to hide or lock a .ttf/.otf file from the
  browser, stop font downloads via DevTools, keep text sharp at any size, or
  build a font preview / type tester that never ships the font to the client.
license: MIT
compatibility: >-
  Any agent runtime; server-side JavaScript on Node.js 18+ or Bun 1.x.
  Framework-agnostic through a Request/Response adapter. Requires Node runtime
  for opentype.js, Sharp, and subset-font; not compatible with Edge-only
  runtimes.
metadata:
  audience: web-developers
  categories: "fonts,security,image-generation"
---

# Instruksi Operasional

Agent **WAJIB** memanggil tool `context7` dalam kondisi berikut:

1. Saat membutuhkan dokumentasi teknis atau referensi API terbaru dari pustaka eksternal.
2. Saat menganalisis struktur kode atau konteks proyek yang membutuhkan pencarian semantik mendalam.

### Ketentuan Pemanggilan:

- Gunakan query yang spesifik dan terfokus pada topik teknis.
- Contoh format aksi: Panggil `context7` dengan parameter query yang relevan, lalu saring hasilnya sebelum disajikan kepada pengguna.
- Jangan memanggil `context7` untuk pertanyaan umum yang tidak memerlukan referensi eksternal/konteks kode.

# Server-Side Font Protection

Tiga pola untuk menampilkan teks dengan font komersial/privat tanpa pernah
mengirim file font ke browser. Font tetap di server; browser hanya menerima
**vektor**, **gambar**, atau **subset font sementara**.

Untuk matriks kompatibilitas lintas runtime, framework, format font, dan
browser, lihat [README.md](README.md).

## Kapan skill ini dipakai

- "Font-nya jangan bisa didownload lewat Network tab."
- "Preview font harus tetap tajam di semua ukuran layar."
- "Di Inspect Element jangan kelihatan file font / `@font-face`."
- "Bikin type tester / font preview yang font-nya locked."
- "Convert text to SVG path / render text to image / subset font on the fly."

## Pilih pendekatan

| Kebutuhan                                          | Pendekatan                   | Output                     | Kekuatan                                   | Kelemahan                                             |
| -------------------------------------------------- | ---------------------------- | -------------------------- | ------------------------------------------ | ----------------------------------------------------- |
| Teks tajam, ringan, bisa diwarnai CSS              | 1. SVG path (opentype.js)    | `image/svg+xml`            | Tajam tanpa batas ukuran, kecil, cacheable | Outline glyph bisa diimpor balik jadi font            |
| Proteksi maksimal, teks asli tidak perlu keluar    | 2. Raster (Satori + Sharp)   | `image/png` / `image/webp` | Font dan outline tidak pernah keluar       | Tidak selectable, perlu `alt`/aria, resolusi terbatas |
| Teks asli tetap terkirim (bisa di-select, di-copy) | 3. Subset font (subset-font) | `font/woff2`               | Teks asli & aksesibel, file kecil          | Huruf yang dipakai bisa dipanen lewat banyak request  |

Aturan emas: **file font asli tidak boleh ada di `public/`**, tidak boleh
dirujuk `@font-face` di CSS, dan tidak boleh di-import lewat `next/font`.
Simpan di folder privat, mis. `assets/fonts/`, dan akses hanya lewat Route
Handler dengan allowlist `fontId -> path`.

## Instalasi

installasi tergantung runtime JS yang dipakai. Contoh di bawah untuk Bun, tapi sama saja di Node.js/NPM/Yarn.

```bash
bun add opentype.js satori subset-font
# sharp biasanya sudah ada sebagai dependency Next.js; kalau belum:
bun add sharp
```

- `opentype.js` — parse TTF/OTF/WOFF, `font.getPath()` → `path.toPathData()`.
- `satori` — HTML/JSX → SVG string, mendukung TTF/OTF/WOFF/WOFF2 dari Buffer.
- `sharp` — SVG Buffer → PNG/WebP.
- `subset-font` — HarfBuzz (WASM) subsetter; output `sfnt`/`woff`/`woff2`.

## Route handler: aturan wajib

Semua contoh memakai Next.js App Router (`app/api/**/route.ts`) di Node runtime.
Route Handler tidak di-cache by default, jadi jangan tambahkan
`export const dynamic = 'force-static'` untuk endpoint yang menerima input user.

1. **Allowlist font.** Jangan pernah menerima path/nama file dari client.
   ```ts
   const FONTS = {
     "inter-display": "assets/fonts/Inter-Display.otf",
   } as const;
   ```
2. **Cache font yang sudah diparse.** `opentype.parse()` mahal. Simpan di module
   scope (atau `globalThis` saat dev agar tidak re-parse tiap HMR).
3. **Validasi teks.** Batasi panjang (mis. 200–500 karakter), buang karakter
   kontrol, tolak string kosong. Ini mencegah DoS lewat path raksasa atau subset
   raksasa.
4. **Rate limit.** Terutama untuk endpoint subset dan raster.
5. **Cache-Control.** SVG/gambar/subset boleh `public, max-age=31536000,
immutable` bila di-key dengan hash teks + font + opsi. Untuk teks sensitif,
   pakai `private, no-store`.
6. **Deploy.** File font harus ikut ter-trace. Di `next.config.ts`:
   ```ts
   const nextConfig = {
     outputFileTracingIncludes: {
       "/api/font/**": ["./assets/fonts/**"],
     },
     serverExternalPackages: ["sharp", "subset-font"],
   };
   ```
7. **Node runtime.** `opentype.js`, `sharp`, dan WASM subsetter tidak jalan di
   Edge. Jangan set `export const runtime = "edge"`.

## Pola 1 — Teks → SVG path (opentype.js)

```ts
const font = opentype.parse(toArrayBuffer(await readFile(fontPath)));
const path = font.getPath(text, 0, 0, fontSize, { kerning: true });
const bbox = path.getBoundingBox();
const advance = font.getAdvanceWidth(text, fontSize, { kerning: true });

const d = path.toPathData({
  decimalPlaces: 2,
  optimize: true,
  flipY: true,
  flipYBase: bbox.y2, // font y-axis naik, SVG y-axis turun
});

const svg = `<svg xmlns="http://www.w3.org/2000/svg"
  viewBox="${bbox.x1} 0 ${Math.max(advance, bbox.x2) - bbox.x1} ${bbox.y2 - bbox.y1}"
  role="img" aria-label="${escapeXml(text)}">
  <path d="${d}" fill="currentColor"/>
</svg>`;
```

Kembalikan `new Response(svg, { headers: { "Content-Type": "image/svg+xml" } })`,
lalu di client pakai `<img src="/api/font/svg?text=...">`, atau `fetch` +
inline SVG bila perlu mewarisi `currentColor`. Detail lengkap, termasuk opsi
`flipY`, viewBox, cache, dan kebocoran outline: `references/01-svg-paths.md`.

## Pola 2 — Teks → PNG/WebP (Satori + Sharp)

```ts
const svg = await satori(
  React.createElement(
    "div",
    {
      style: {
        display: "flex",
        width: "100%",
        height: "100%",
        alignItems: "center",
        justifyContent: "center",
        background: bg,
        color: fg,
        fontSize,
        fontFamily: "Preview",
      },
    },
    text,
  ),
  {
    width,
    height,
    fonts: [
      { name: "Preview", data: fontBuffer, weight: 400, style: "normal" },
    ],
  },
);

const png = await sharp(Buffer.from(svg)).png().toBuffer();
return new Response(png, { headers: { "Content-Type": "image/png" } });
```

`sharp(svg).webp({ quality: 90 })` untuk WebP. Ukuran canvas dihitung dari
`font.getAdvanceWidth()` supaya tidak ada ruang kosong berlebih. Detail:
`references/02-image-render.md`.

## Pola 3 — Dynamic font subsetting (subset-font)

```ts
const subset = await subsetFont(fontBuffer, text, {
  targetFormat: "woff2",
  // keepFeatures: [], noHinting: true,
});
return new Response(subset, {
  headers: {
    "Content-Type": "font/woff2",
    "Cache-Control": "private, max-age=60",
  },
});
```

Di client:

```ts
const res = await fetch(`/api/font/subset?text=${encodeURIComponent(text)}`);
const url = URL.createObjectURL(await res.blob());
const face = new FontFace("PreviewSubset", `url(${url})`);
await face.load();
document.fonts.add(face);
el.style.fontFamily = "PreviewSubset";
```

Setiap perubahan teks menghasilkan file baru yang hanya berisi huruf yang
diketik. Detail, cache, dan batasan keamanan: `references/03-font-subsetting.md`.

## Referensi

- `references/01-svg-paths.md` — route handler SVG lengkap, viewBox, flipY, cache, ekstraksi outline.
- `references/02-image-render.md` — Satori + Sharp, ukuran dinamis, WebP, alt text, caching, batas resolusi.
- `references/03-font-subsetting.md` — subset-font, FontFace di client, cache ETag, panen glyph & mitigasi.
- `references/04-security-and-performance.md` — threat model jujur, checklist produksi, tuning performa.

## Jangan klaim lebih dari yang diberikan

- SVG path **bukan DRM**: outline tiap glyph ada di atribut `d` dan bisa
  diimpor balik (`opentype.Glyph.fromSVG`) menjadi font baru. Ini menaikkan
  biaya ekstraksi, bukan menghilangkannya.
- Subset font **bukan enkripsi**: penyerang bisa memanggil endpoint berkali-kali
  dengan karakter berbeda lalu menggabungkan hasilnya. Mitigasi: rate limit,
  batas panjang, deteksi pola, dan jangan pakai subset untuk teks yang sangat
  sensitif.
- Raster adalah satu-satunya pola yang tidak mengirim outline, tapi tetap bisa
  di-OCR. Perlindungan font ≠ perlindungan teks.
