# Security, Performance, dan Checklist Produksi

## 1. Threat model jujur

| Penyerang | SVG path | Raster PNG/WebP | Subset font |
|---|---|---|---|
| User biasa buka DevTools | Tidak melihat `@font-face`/file font | Tidak melihat apa pun selain gambar | Melihat file `.woff2`, tapi hanya berisi huruf yang diketik |
| Menyalin `d` dari Inspect Element | Bisa; outline bisa diimpor balik | Tidak ada outline | Bisa, untuk glyph yang ada |
| Harvesting lewat banyak request | Bisa mengumpulkan path per karakter | Bisa dikumpulkan sebagai gambar lalu di-OCR | Bisa dengan request berulang; perlu rate limit |
| Rekonstruksi font utuh | Mungkin, dengan usaha | Sangat sulit (perlu OCR + tracing) | Mungkin, dengan usaha dan banyak request |

Kesimpulan:

- **SVG** = anti-casual, teks tetap tajam dan ringan.
- **Raster** = proteksi terkuat yang praktis, tetapi kehilangan seleksi teks.
- **Subset** = kompromi: teks asli & aksesibel, file kecil, tapi bukan DRM.

Jangan pernah mengklaim "font tidak bisa dicuri" untuk salah satu pola.

## 2. Checklist wajib

### Penyimpanan font

- [ ] Font ada di `assets/fonts/` atau folder privat lain, **bukan** `public/`.
- [ ] Tidak ada `@font-face` yang menunjuk file asli di CSS.
- [ ] Tidak memakai `next/font/local` untuk font yang dilindungi (Next akan
      menyalin font ke output publik).
- [ ] Font tidak di-commit ke repo publik bila lisensinya melarang.
- [ ] Nama file tidak membocorkan nama family/versi di response/error.

### Endpoint

- [ ] Allowlist `fontId → path`; tidak ada path dari client.
- [ ] Validasi panjang teks, tolak string kosong dan karakter kontrol.
- [ ] Validasi `size`, `scale`, `color`, `format` dengan clamp/allowlist.
- [ ] Tolak glyph yang tidak ada (`.notdef`) atau beri fallback yang jelas.
- [ ] Rate limit per IP/session.
- [ ] `Cache-Control` sesuai sensitivitas teks.
- [ ] `ETag` + dukungan `If-None-Match` (304).
- [ ] `X-Content-Type-Options: nosniff`.
- [ ] Tidak ada CORS `*` kecuali memang endpoint publik.
- [ ] Error tidak memuat stack trace, path file, atau isi font.

### Deploy

- [ ] Node runtime, bukan Edge.
- [ ] `outputFileTracingIncludes` mencakup folder font.
- [ ] `serverExternalPackages` untuk `sharp` dan `subset-font` bila perlu.
- [ ] Ukuran bundle serverless muat (font + WASM + sharp).
- [ ] Uji di production build, bukan hanya `next dev`.

## 3. Header response yang disarankan

```ts
const headers = {
  "Content-Type": "image/svg+xml; charset=utf-8", // atau image/png, font/woff2
  "Cache-Control": "public, max-age=31536000, immutable",
  ETag: etag,
  "X-Content-Type-Options": "nosniff",
  // Kalau SVG di-inline dan bisa mengandung konten user:
  "Content-Security-Policy": "default-src 'none'; style-src 'unsafe-inline'; sandbox",
};
```

- Untuk endpoint yang menerima teks sensitif: `Cache-Control: private, no-store`
  dan jangan pakai ETag yang bisa dijadikan oracle isi teks.
- `Content-Disposition: inline` untuk gambar/SVG; untuk subset font,
  `inline; filename="preview.woff2"` membantu DevTools tetapi tidak wajib.

## 4. Performa

| Masalah | Solusi |
|---|---|
| `opentype.parse` di setiap request | Cache `Promise<Loaded>` per font di module scope / `globalThis` |
| Rasterisasi per ketikan | Debounce 150–300 ms + `AbortController` + in-flight dedupe |
| Subset WASM per request | Cache LRU berdasarkan `fontId:text`, warm-up saat startup |
| Payload besar | WOFF2 untuk subset, WebP untuk raster, SVG untuk vektor |
| Canvas raster kebesaran | Batas `MAX_WIDTH`, `scale ≤ 3`, `limitInputPixels` |
| Font weight banyak | Hanya load weight yang dipakai; instance variable font |
| Cold start | Lazy import `sharp`/`subset-font`, atau route terpisah per pola |

Aturan praktis:

- SVG path: paling murah, aman untuk preview real-time.
- Raster: 5–50 ms per gambar tergantung ukuran; cache agresif.
- Subset: 10–100 ms per teks baru (WASM + shaping); cache dan debounce.

## 5. Integrasi Next.js 16

- Route Handler tidak di-cache by default; hanya `GET` yang bisa di-cache, dan
  jangan aktifkan `force-static` untuk input dinamis.
- `params` di route dinamis adalah `Promise`; gunakan `await params`.
- Gunakan `new URL(request.url).searchParams` untuk query.
- Konfigurasi:

```ts
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  outputFileTracingIncludes: {
    "/api/font/**": ["./assets/fonts/**"],
  },
  serverExternalPackages: ["sharp", "subset-font"],
};

export default nextConfig;
```

- Jangan set `export const runtime = "edge"` pada route yang memakai
  `opentype.js`, `sharp`, atau `subset-font`.
- Untuk pengembangan, `globalThis` cache mencegah re-parse saat HMR.

## 6. Uji cepat

```bash
# SVG
curl -s "http://localhost:3000/api/font/svg?font=inter-display&text=Halo&size=96" | head -c 400

# PNG
curl -s -o /tmp/preview.png -w "%{content_type} %{size_download}\n" \
  "http://localhost:3000/api/font/image?font=inter-display&text=Halo&size=96"

# WebP
curl -s -H "Accept: image/webp" -o /tmp/preview.webp -w "%{content_type}\n" \
  "http://localhost:3000/api/font/image?font=inter-display&text=Halo"

# Subset
curl -s -o /tmp/subset.woff2 -w "%{content_type} %{size_download}\n" \
  "http://localhost:3000/api/font/subset?font=inter-display&text=Halo"
```

Verifikasi:

- `file /tmp/preview.png` → PNG image data.
- `file /tmp/subset.woff2` → Web Open Font Format (Version 2).
- Buka DevTools → Network: tidak ada request ke file `.ttf`/`.otf` asli.
- Inspect Element pada SVG: hanya `<path>`, tidak ada `<text>` atau `@font-face`.
- Ganti teks dan pastikan respons berubah / ETag berubah.

## 7. Kesalahan umum

1. **Font di `public/`** — siapa pun bisa mengunduhnya; semua pola jadi sia-sia.
2. **Mengandalkan `user-select: none`** — itu hanya mencegah seleksi, bukan
   pengambilan file.
3. **Mengira SVG = aman** — outline ada di `d`.
4. **Mengira subset = enkripsi** — bisa dipanen.
5. **Tidak membatasi panjang teks** — path/subset bisa membengkak dan memakan
   CPU/memori.
6. **Meng-cache teks sensitif secara publik** — kebocoran lewat cache/CDN.
7. **Memakai Edge runtime** — `sharp`/WASM gagal.
8. **Lupa `outputFileTracingIncludes`** — font tidak ditemukan di production.
9. **Menambahkan `FontFace` baru tanpa menghapus yang lama** — memory leak dan
   teks bisa memakai subset yang salah.
10. **Tidak menguji dengan font yang punya ligatur/kerning** — hasil SVG dan
    raster bisa berbeda dari ekspektasi.
