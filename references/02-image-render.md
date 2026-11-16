# Pola 2 — Teks → PNG / WebP (Satori + Sharp)

Alur: teks + font Buffer → Satori menghasilkan **SVG string** (teks sudah
dikonversi jadi path oleh Satori) → Sharp merasterisasi SVG menjadi PNG/WebP.
Browser hanya menerima gambar; file font dan outline vektor tidak pernah
sampai ke client.

## 1. Route handler

`app/api/font/image/route.tsx` (harus `.tsx` kalau memakai JSX):

```tsx
import { createHash } from "node:crypto";
import sharp from "sharp";
import satori from "satori";
import { FONTS, loadFont, type FontId } from "@/lib/fonts";

const MAX_TEXT = 120;
const MAX_WIDTH = 1600;
const MAX_SCALE = 3;

export async function GET(request: Request) {
  const url = new URL(request.url);

  const fontId = url.searchParams.get("font") as FontId | null;
  if (!fontId || !(fontId in FONTS)) {
    return Response.json({ error: "Unknown font" }, { status: 400 });
  }

  const text = (url.searchParams.get("text") ?? "").normalize("NFC").trim();
  if (!text || text.length > MAX_TEXT) {
    return Response.json({ error: "Invalid text" }, { status: 400 });
  }

  const fontSize = Math.min(240, Math.max(12, Number(url.searchParams.get("size")) || 72));
  const scale = Math.min(MAX_SCALE, Math.max(1, Number(url.searchParams.get("scale")) || 2));
  const bg = url.searchParams.get("bg") ?? "#ffffff";
  const fg = url.searchParams.get("fg") ?? "#111111";
  const pad = Math.round(fontSize * 0.35);

  const { buffer, font } = await loadFont(fontId);

  // Ukur teks memakai advance width supaya canvas tidak kebesaran/kekecilan.
  const advance = font.getAdvanceWidth(text, fontSize, { kerning: true });
  const width = Math.min(MAX_WIDTH, Math.ceil(advance + pad * 2));
  const height = Math.ceil(fontSize * 1.4 + pad * 2);

  // Render pada resolusi lebih tinggi, lalu client menampilkan pada ukuran CSS.
  const renderWidth = Math.ceil(width * scale);
  const renderHeight = Math.ceil(height * scale);

  const svg = await satori(
    <div
      style={{
        display: "flex",
        width: "100%",
        height: "100%",
        alignItems: "center",
        justifyContent: "center",
        background: bg,
        color: fg,
        fontFamily: "Preview",
        fontSize: fontSize * scale,
        whiteSpace: "pre",
        lineHeight: 1.2,
      }}
    >
      {text}
    </div>,
    {
      width: renderWidth,
      height: renderHeight,
      fonts: [{ name: "Preview", data: buffer, weight: 400, style: "normal" }],
    },
  );

  const accept = request.headers.get("accept") ?? "";
  const useWebp = accept.includes("image/webp");

  let image: Buffer;
  if (useWebp) {
    image = await sharp(Buffer.from(svg), { limitInputPixels: 40_000_000 })
      .webp({ quality: 92, effort: 4 })
      .toBuffer();
  } else {
    image = await sharp(Buffer.from(svg), { limitInputPixels: 40_000_000 })
      .png({ compressionLevel: 9 })
      .toBuffer();
  }

  const etag = `"${createHash("sha256").update(image).digest("base64url")}"`;
  if (request.headers.get("if-none-match") === etag) {
    return new Response(null, { status: 304, headers: { ETag: etag } });
  }

  return new Response(image, {
    headers: {
      "Content-Type": useWebp ? "image/webp" : "image/png",
      "Content-Length": String(image.byteLength),
      "Cache-Control": "public, max-age=31536000, immutable",
      ETag: etag,
      "X-Content-Type-Options": "nosniff",
    },
  });
}
```

Kalau file route tidak ingin memakai JSX, ganti elemen dengan
`React.createElement("div", { style: {...} }, text)` dan import `React` dari
`react`. Satori juga menerima HTML string lewat `satori-html`, tapi JSX lebih
mudah di-type.

## 2. Menentukan ukuran canvas

- `font.getAdvanceWidth(text, fontSize, { kerning: true })` menghitung lebar
  layout termasuk spasi; ini lebih tepat daripada bbox untuk teks satu baris.
- Tambahkan padding agar overhang italic dan descender tidak terpotong.
- Untuk teks panjang, jangan biarkan `advance` menentukan canvas tanpa batas.
  Tetapkan `MAX_WIDTH`, lalu bungkus teks. Satori mendukung `flexWrap: "wrap"`
  dan `lineClamp`, tetapi tinggi canvas harus dihitung dari jumlah baris yang
  diharapkan. Alternatif yang lebih aman: pecah teks per baris secara manual
  dan hitung `height = lines.length * fontSize * lineHeight + padding * 2`.
- Batas piksel: `width * height * scale^2` harus dibatasi (mis. ≤ 40 juta
  piksel) agar Sharp tidak kehabisan memori. `limitInputPixels` adalah jaring
  pengaman terakhir.
- `scale` 2 sudah cukup untuk layar retina; 3 hanya untuk teks kecil.

## 3. PNG vs WebP

| Format | Kapan dipakai | Catatan |
|---|---|---|
| PNG | teks kecil, butuh alpha, kompatibilitas maksimal | lossless, ukuran lebih besar |
| WebP | teks besar/foto latar, bandwidth sensitif | `quality: 90–95`; teks tajam tetap aman di lossy tinggi |
| AVIF | opsional | encoding lebih lambat; hindari untuk request per ketikan |

Negosiasi isi `Accept` seperti contoh di atas, atau sediakan query `?format=webp`
yang divalidasi dengan allowlist.

## 4. Font & Satori

- Satori mendukung TTF, OTF, WOFF, WOFF2 dari `Buffer`/`ArrayBuffer`.
- `name` di `fonts[]` harus sama dengan `fontFamily` di style.
- Kirim hanya weight/style yang dipakai. Setiap font diparse dan di-cache oleh
  Satori berdasarkan Buffer; menambah banyak weight memperbesar memori.
- Font variable: Satori memakai instance default. Untuk weight tertentu,
  subset/instance dulu dengan `subset-font` (`variationAxes: { wght: 700 }`).
- Satori mengeluarkan `<path>` untuk teks, bukan `<text>`. Jadi SVG antara
  (sebelum Sharp) juga sudah tidak membocorkan file font, hanya outline.

## 5. Client + aksesibilitas

```tsx
<img
  src={`/api/font/image?font=inter-display&size=96&text=${encodeURIComponent(text)}`}
  alt={text}
  width={960}
  height={160}
  style={{ maxWidth: "100%", height: "auto" }}
/>
```

- `alt` wajib supaya teks tetap terbaca screen reader dan SEO.
- Karena teks tidak selectable, sediakan kontrol copy di UI bila pengguna butuh
  menyalin teks.
- Jangan set `width`/`height` sembarangan; kirim dimensi asli lewat header
  `X-Image-Width`/`X-Image-Height` atau endpoint metadata bila layout butuh
  rasio yang tepat.
- Debounce input 150–300 ms dan batalkan request lama dengan `AbortController`.
  Rasterisasi lebih mahal daripada SVG path.

## 6. Cache

Rasterisasi mahal, jadi cache sangat penting:

1. **HTTP cache** — key sudah unik (font + teks + size + warna + scale), jadi
   `immutable` aman.
2. **Disk cache** — simpan hasil di `.cache/font-images/<sha256>.webp`. Pada
   serverless, pakai object storage (S3/R2) atau cache layer platform.
3. **In-flight dedupe** — `Map<string, Promise<Buffer>>` agar dua request
   identik tidak merender dua kali.

## 7. Jebakan umum

- `sharp` tidak bisa merender teks SVG tanpa font sistem; karena Satori sudah
  mengubah teks jadi path, masalah ini tidak muncul. Jangan memberi Sharp SVG
  yang masih berisi `<text>`.
- Jangan pakai `dangerouslySetInnerHTML` untuk SVG hasil Satori di client;
  cukup kirim sebagai gambar.
- Di Edge runtime, `sharp` tidak tersedia. Tetap di Node runtime.
- Saat deploy, pastikan font ikut ter-trace (`outputFileTracingIncludes`) dan
  `sharp` tidak di-bundle ulang (`serverExternalPackages: ["sharp"]`).
