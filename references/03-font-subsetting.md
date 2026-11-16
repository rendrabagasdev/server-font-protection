# Pola 3 — Dynamic Font Subsetting (subset-font)

Server memotong file font sehingga hanya berisi glyph untuk huruf yang sedang
diketik pengunjung. Browser tetap menerima file font asli (bisa di-select,
di-copy, dan di-cache), tetapi file itu tidak bisa dipakai sebagai font utuh.
Kalau diambil lewat Network tab, isinya hanya beberapa huruf dari teks saat itu.

## 1. Route handler

`app/api/font/subset/route.ts`:

```ts
import { createHash } from "node:crypto";
import subsetFont from "subset-font";
import { FONTS, loadFont, type FontId } from "@/lib/fonts";

const MAX_TEXT = 300;
const MAX_CACHE_ENTRIES = 200;

type Entry = { body: Buffer; etag: string };
const cache = new Map<string, Entry>();

function getCached(key: string) {
  const hit = cache.get(key);
  if (!hit) return null;
  // Refresh urutan LRU.
  cache.delete(key);
  cache.set(key, hit);
  return hit;
}

function setCached(key: string, entry: Entry) {
  cache.set(key, entry);
  while (cache.size > MAX_CACHE_ENTRIES) {
    const oldest = cache.keys().next().value;
    if (oldest === undefined) break;
    cache.delete(oldest);
  }
}

export async function GET(request: Request) {
  const url = new URL(request.url);

  const fontId = url.searchParams.get("font") as FontId | null;
  if (!fontId || !(fontId in FONTS)) {
    return Response.json({ error: "Unknown font" }, { status: 400 });
  }

  const text = (url.searchParams.get("text") ?? "").normalize("NFC");
  if (!text.trim() || text.length > MAX_TEXT) {
    return Response.json({ error: "Invalid text" }, { status: 400 });
  }

  const key = `${fontId}:${text}`;
  const cached = getCached(key);
  if (cached) return respond(request, cached);

  const { buffer } = await loadFont(fontId);

  const body = await subsetFont(buffer, text, {
    targetFormat: "woff2",
    // Ligature tetap bekerja; hapus opsi ini kalau ingin file lebih kecil
    // tetapi rela kehilangan liga/rlig.
    // keepFeatures: ["liga", "rlig"],
    noHinting: true,
  });

  const entry: Entry = {
    body,
    etag: `"${createHash("sha256").update(body).digest("base64url")}"`,
  };
  setCached(key, entry);
  return respond(request, entry);
}

function respond(request: Request, entry: Entry) {
  if (request.headers.get("if-none-match") === entry.etag) {
    return new Response(null, { status: 304, headers: { ETag: entry.etag } });
  }
  return new Response(entry.body, {
    headers: {
      "Content-Type": "font/woff2",
      "Content-Length": String(entry.body.byteLength),
      "Cache-Control": "private, max-age=300",
      ETag: entry.etag,
      "X-Content-Type-Options": "nosniff",
    },
  });
}
```

Catatan:

- `subsetFont(buffer, text, options)` mengembalikan `Promise<Buffer>`.
- `targetFormat`: `"sfnt"` | `"woff"` | `"woff2"`. WOFF2 paling kecil dan
  didukung semua browser modern.
- `noHinting: true` memangkas instruksi hinting (tidak berguna di layar modern).
- `keepFeatures: []` menghapus semua layout feature dan membuat file lebih
  kecil, tetapi ligatur/kerning berbasis GSUB bisa hilang.
- `variationAxes` untuk font variable, mis. `{ wght: { min: 400, max: 700 } }`.
- `keepAllGlyphs: true` hanya untuk instancing, bukan subsetting.
- Jangan mengirim `text` kosong; subset-font bisa menghasilkan font tanpa glyph
  yang berguna.

## 2. Client: FontFace API

```tsx
"use client";
import { useEffect, useRef, useState } from "react";

const FAMILY = "PreviewSubset";

export function SubsetPreview({ text }: { text: string }) {
  const [ready, setReady] = useState(false);
  const faceRef = useRef<FontFace | null>(null);
  const urlRef = useRef<string | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    const timer = setTimeout(async () => {
      try {
        const res = await fetch(
          `/api/font/subset?font=inter-display&text=${encodeURIComponent(text)}`,
          { signal: controller.signal },
        );
        if (!res.ok) return;

        const blob = await res.blob();
        const url = URL.createObjectURL(blob);
        const face = new FontFace(FAMILY, `url(${url})`, { display: "block" });
        await face.load();

        // Bersihkan face & blob lama agar tidak menumpuk di memori.
        if (faceRef.current) document.fonts.delete(faceRef.current);
        if (urlRef.current) URL.revokeObjectURL(urlRef.current);

        document.fonts.add(face);
        faceRef.current = face;
        urlRef.current = url;
        setReady(true);
      } catch (error) {
        if ((error as Error).name !== "AbortError") console.error(error);
      }
    }, 200); // debounce

    return () => {
      controller.abort();
      clearTimeout(timer);
    };
  }, [text]);

  return (
    <div
      style={{
        fontFamily: ready ? FAMILY : "system-ui",
        fontSize: 96,
        // Hindari FOIT: tampilkan fallback sampai subset siap.
        visibility: ready ? "visible" : "hidden",
      }}
    >
      {text}
    </div>
  );
}
```

Pitfall:

- Nama family yang sama untuk face berbeda akan menumpuk di `document.fonts`.
  Simpan referensi `FontFace` lama dan `document.fonts.delete()` sebelum
  menambahkan yang baru.
- `URL.revokeObjectURL` hanya setelah `face.load()` selesai; kalau terlalu awal,
  font gagal dimuat.
- Saat teks berubah, ada jeda loading. Pakai `display: "block"` atau tampilkan
  teks fallback dengan transisi halus; jangan biarkan layout melompat.
- Untuk teks yang sangat panjang, subset bisa mendekati ukuran font asli. Batasi
  `MAX_TEXT`.

## 3. Cache

- **HTTP cache**: `private, max-age=300` cukup untuk mencegah request berulang
  saat pengguna mengetik mundur/maju. Kalau teks tidak sensitif, boleh
  `public, max-age=31536000, immutable` karena key sudah berisi teks.
- **LRU di memori**: contoh di atas membatasi 200 entri. Ukuran subset biasanya
  beberapa KB sampai puluhan KB, jadi aman. Di serverless, memori tidak dibagi
  antar instance; cache disk/object storage lebih efektif.
- **ETag/304**: penting karena setiap perubahan teks memicu request baru.

## 4. Keamanan: panen glyph

Subset bukan enkripsi. Penyerang yang gigih bisa:

1. Memanggil endpoint dengan `text=a`, `text=b`, ... untuk semua karakter.
2. Menggabungkan hasilnya, atau mengekstrak outline tiap glyph dari font subset.
3. Menyusun font lengkap.

Mitigasi berlapis:

- **Rate limit per IP/session** (mis. 30 request/menit, burst 10).
- **Batas panjang teks** dan **batas variasi** (jangan izinkan query acak
  berulang dengan string 1 karakter).
- **Minimal panjang teks** untuk endpoint publik (mis. 2–3 karakter) bila
  memungkinkan secara UX.
- **Deteksi pola**: blokir rentetan request yang menaikkan cakupan karakter
  (a, ab, abc, ...) atau yang meminta karakter kontrol/private-use.
- **Jangan tampilkan jumlah glyph / metadata font** di response.
- Untuk font yang benar-benar sensitif, gunakan Pola 2 (raster), bukan subset.

Batas teknis: subset-font melakukan glyph closure untuk layout (ligatur), jadi
file bisa berisi beberapa glyph tambahan di luar huruf yang diketik. Itu wajar
dan bukan bug.

## 5. Deploy & performa

- `subset-font` memakai HarfBuzz via WebAssembly. Jalankan di Node runtime,
  bukan Edge.
- WASM perlu ikut ter-trace. Kalau build serverless gagal menemukan `.wasm`,
  tambahkan:
  ```ts
  serverExternalPackages: ["subset-font"],
  outputFileTracingIncludes: {
    "/api/font/subset": ["./node_modules/harfbuzzjs/**", "./assets/fonts/**"],
  },
  ```
- Cache `buffer` font asli (sudah dilakukan `loadFont`) agar tidak membaca disk
  tiap request.
- Subset pertama per proses membayar biaya inisialisasi WASM. Hangatkan saat
  startup bila traffic tinggi.
- Ukuran output: WOFF2 + `noHinting` biasanya 2–10 KB untuk satu kalimat.

## 6. Alternatif

- **`pyftsubset` (fonttools)** — lebih cepat untuk batch, tetapi butuh Python
  di runtime. Cocok untuk pre-generate subset per unicode-range saat build.
- **fontkit** — subsetting di Node tanpa WASM, tetapi API lebih manual.
- **Pre-subset statis** — kalau alfabet diketahui (mis. Latin), bagi font jadi
  beberapa file `unicode-range` dan sajikan hanya yang diperlukan. Ini bukan
  dynamic subsetting, tapi sering cukup dan lebih murah.
