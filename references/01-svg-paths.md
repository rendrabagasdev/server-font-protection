# Pola 1 — Teks → SVG Vector Path (opentype.js)

Browser tidak pernah menerima file font. Server memparse font, mengubah teks
menjadi `<path d="...">`, dan mengirim SVG. Hasilnya tajam di semua ukuran
karena SVG diskalakan sebagai vektor, dan Inspect Element hanya menampilkan
kode garis.

## 1. Loader font (cache + allowlist)

`lib/fonts.ts`:

```ts
import { readFile } from "node:fs/promises";
import path from "node:path";
import opentype from "opentype.js";

export const FONTS = {
  "inter-display": "assets/fonts/Inter-Display.otf",
  "serif-editorial": "assets/fonts/EditorialNew-Regular.ttf",
} as const;

export type FontId = keyof typeof FONTS;

// Simpan Buffer asli (untuk Satori/subset) dan Font hasil parse (untuk path).
type ParsedFont = ReturnType<typeof opentype.parse>;
type Loaded = { buffer: Buffer; font: ParsedFont };

// globalThis supaya cache bertahan saat HMR di dev.
const globalCache = globalThis as unknown as {
  __fontCache?: Map<string, Promise<Loaded>>;
};
const cache = (globalCache.__fontCache ??= new Map());

export function loadFont(id: FontId): Promise<Loaded> {
  const existing = cache.get(id);
  if (existing) return existing;

  const promise = (async () => {
    const file = path.join(process.cwd(), FONTS[id]);
    const buffer = await readFile(file);
    // opentype.parse butuh ArrayBuffer, bukan Buffer.
    const arrayBuffer = buffer.buffer.slice(
      buffer.byteOffset,
      buffer.byteOffset + buffer.byteLength,
    );
    const font = opentype.parse(arrayBuffer);
    return { buffer, font };
  })();

  cache.set(id, promise);
  // Kalau gagal, jangan simpan promise yang rejected.
  promise.catch(() => cache.delete(id));
  return promise;
}
```

Catatan:

- Jangan taruh font di `public/`. Simpan di `assets/fonts/` (di luar `public`).
- `opentype.parse` menerima `ArrayBuffer`. Kalau memberi `Buffer` langsung,
  offset bisa salah untuk buffer pool Node.
- WOFF2 tidak didukung opentype.js secara langsung. Dekompres dulu (mis.
  `wawoff2`) atau simpan sumber sebagai TTF/OTF/WOFF.

## 2. Route handler

`app/api/font/svg/route.ts`:

```ts
import { createHash } from "node:crypto";
import { FONTS, loadFont, type FontId } from "@/lib/fonts";

const MAX_TEXT = 200;
const MAX_FONT_SIZE = 400;
const MIN_FONT_SIZE = 8;

function escapeXml(value: string) {
  return value.replace(/[<>&"']/g, (ch) => {
    switch (ch) {
      case "<": return "&lt;";
      case ">": return "&gt;";
      case "&": return "&amp;";
      case '"': return "&quot;";
      default: return "&apos;";
    }
  });
}

function cleanText(raw: string) {
  // Buang karakter kontrol kecuali newline & tab, normalisasi unicode.
  const text = raw.normalize("NFC").replace(/[\u0000-\u0008\u000B\u000C\u000E-\u001F\u007F]/g, "");
  if (!text.trim()) return null;
  if (text.length > MAX_TEXT) return null;
  return text;
}

export async function GET(request: Request) {
  const url = new URL(request.url);

  const fontId = url.searchParams.get("font") as FontId | null;
  if (!fontId || !(fontId in FONTS)) {
    return Response.json({ error: "Unknown font" }, { status: 400 });
  }

  const text = cleanText(url.searchParams.get("text") ?? "");
  if (!text) {
    return Response.json({ error: "Invalid text" }, { status: 400 });
  }

  const fontSize = Math.min(
    MAX_FONT_SIZE,
    Math.max(MIN_FONT_SIZE, Number(url.searchParams.get("size")) || 72),
  );

  const colorParam = url.searchParams.get("color") ?? "#111111";
  const color = /^#[0-9a-fA-F]{3,8}$/.test(colorParam) ? colorParam : "#111111";

  const { font } = await loadFont(fontId);

  // Tolak karakter yang tidak ada di font (glyph .notdef).
  for (const ch of text) {
    if (ch === "\n" || ch === "\t") continue;
    if (font.charToGlyph(ch).index === 0) {
      return Response.json({ error: `Missing glyph: ${ch}` }, { status: 422 });
    }
  }

  // Untuk multi-baris: pecah, render per baris, gabungkan bbox.
  const lines = text.split("\n");
  const lineHeight = fontSize * 1.2;

  // Pass 1: render tiap baris & kumpulkan bbox global.
  const rendered = lines
    .map((line, index) => {
      if (!line) return null;
      const path = font.getPath(line, 0, index * lineHeight, fontSize, {
        kerning: true,
        features: { liga: true },
      });
      return { path, bbox: path.getBoundingBox() };
    })
    .filter((item): item is NonNullable<typeof item> => item !== null);

  let minX = Infinity;
  let maxX = -Infinity;
  let minY = Infinity;
  let maxY = -Infinity;
  for (const { bbox } of rendered) {
    minX = Math.min(minX, bbox.x1);
    maxX = Math.max(maxX, bbox.x2);
    minY = Math.min(minY, bbox.y1);
    maxY = Math.max(maxY, bbox.y2);
  }

  // Pass 2: flip semua path dengan basis yang sama agar posisi antar-baris tetap.
  let d = "";
  for (const { path } of rendered) {
    d += path.toPathData({
      decimalPlaces: 2,
      optimize: true,
      flipY: true,
      flipYBase: maxY,
    });
  }

  if (!Number.isFinite(minX) || !Number.isFinite(minY) || !d) {
    return Response.json({ error: "Empty path" }, { status: 422 });
  }

  const width = Math.max(1, maxX - minX);
  const height = Math.max(1, maxY - minY);

  // minX bisa negatif (italic/overhang), jadi viewBox mulai dari minX.
  const svg =
    `<svg xmlns="http://www.w3.org/2000/svg" viewBox="${minX} 0 ${width} ${height}" ` +
    `role="img" aria-label="${escapeXml(text)}">` +
    `<path d="${d}" fill="${color}"/></svg>`;

  const etag = `"${createHash("sha256").update(svg).digest("base64url")}"`;

  if (request.headers.get("if-none-match") === etag) {
    return new Response(null, { status: 304, headers: { ETag: etag } });
  }

  return new Response(svg, {
    headers: {
      "Content-Type": "image/svg+xml; charset=utf-8",
      "Cache-Control": "public, max-age=31536000, immutable",
      ETag: etag,
      "X-Content-Type-Options": "nosniff",
    },
  });
}
```

> `flipYBase: maxY` memakai bbox global supaya semua baris di-flip dengan basis
> yang sama; kalau tiap baris memakai basis sendiri, jarak antar-baris berubah.

## 3. Kenapa viewBox-nya seperti itu

- `font.getPath(text, 0, 0, fontSize)` menaruh baseline di `y = 0`.
- Glyph dengan descender punya `y` negatif; huruf kapital punya `y` positif.
- `toPathData({ flipY: true, flipYBase: B })` memetakan `y' = B - y`, sehingga
  hasilnya berada di `0..height` dengan sumbu Y ke bawah seperti SVG.
- Sumbu X tidak diubah, jadi `viewBox` harus dimulai dari `bbox.x1`, bukan 0.
- `bbox.x1` bisa negatif untuk huruf miring atau overhang.
- Untuk layout teks, lebar advance (`font.getAdvanceWidth`) lebih tepat daripada
  bbox karena menghitung spasi di akhir dan tidak memotong overhang.

Alternatif tanpa `flipY`: keluarkan path apa adanya lalu bungkus dengan
`<g transform="translate(0 ${bbox.y2}) scale(1 -1)">`. Hasil visual sama, tapi
`transform` ikut terlihat di Inspect Element. `flipY` lebih bersih.

## 4. Client

Versi paling sederhana (bisa di-cache browser, aman dari XSS):

```tsx
"use client";
import { useDeferredValue, useState } from "react";

export function SvgPreview() {
  const [text, setText] = useState("Halo dunia");
  const deferred = useDeferredValue(text);

  return (
    <div>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <img
        src={`/api/font/svg?font=inter-display&size=96&text=${encodeURIComponent(deferred)}`}
        alt={deferred}
        style={{ width: "100%", height: "auto" }}
      />
    </div>
  );
}
```

Kalau perlu mewarisi `currentColor`, ambil SVG lewat `fetch` lalu render inline.
Jangan pernah memasukkan input user langsung ke `dangerouslySetInnerHTML` tanpa
sanitasi; di contoh route di atas teks hanya masuk ke `aria-label` yang sudah
di-escape, dan `d`/`fill` dihasilkan server.

## 5. Catatan keamanan

- Path `d` adalah outline asli glyph. Siapa pun bisa menyalinnya dan memakai
  `opentype.Glyph.fromSVG(pathData)` untuk membangun font baru. Anggap ini
  **obfuscation**, bukan DRM.
- Jangan kirim nama file/font path asli di response, error, atau source map.
- Rate limit endpoint karena setiap request melakukan shaping + path generation.
- Kalau teks sensitif, jangan pakai `Cache-Control: public`; gunakan
  `private, no-store` dan hash yang tidak reversibel.
