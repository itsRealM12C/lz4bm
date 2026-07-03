# lz4bm

Decompress BM in LZ4 compressed file

Reverse-engineering notes and usage guide for `lz4bm` — a
self-contained, dependency-free tool that reassembles `LZ4P8`-tagged binary
blobs (commonly pulled from `mmcblkNpM` eMMC/SD partition dumps on embedded
Linux devices) back into a viewable BMP image.

No external libraries, no CDN, no build step. Open the HTML file in any
browser and drop a file on it.

---

## 1. Background

The source file for this analysis (`mmcblk0p25`, 524,288 bytes / 512 KB) was
pulled from an eMMC partition. `mmcblk0p25` decodes as:

- `mmcblk0` — first MMC/eMMC block device (`/dev/mmcblk0`)
- `p25` — partition 25 on that device

This naming convention is standard on Linux (`mmc_block` driver), so any file
named this way is almost certainly a raw `dd`-style partition dump, not a
regular file inside a filesystem.

The partition turned out to hold a single UI bitmap — likely a boot logo,
splash screen, or panel test image — stored in a proprietary compressed
container we're calling **LZ4P8** (from its 5-byte magic).

---

## 2. Container format (`LZ4P8`)

All multi-byte integers are **little-endian**.

| Offset | Size | Field | Notes |
|---|---|---|---|
| `0x00` | 5 bytes | Magic | ASCII `"LZ4P8"` |
| `0x04` | u32 | Total decompressed size | Size of the final reassembled BMP (bytes) |
| `0x08` | u32 | Total compressed size | Header (`0x48` bytes) + sum of all chunk sizes |
| `0x0C` | u32 | Chunk size | Decompressed size of each chunk — `262144` (256 KB), a classic streaming/LZ4 block size |
| `0x10` | u32 | Chunk count | Number of compressed chunks that follow |
| `0x14`–`0x1F` | 12 bytes | Reserved / unknown | Zeroed in the sample file; purpose not yet identified |
| `0x20` | `4 × chunk count` bytes | Chunk size table | One `u32` per chunk: its **compressed** size in bytes |
| `0x48` | variable | Chunk data | Raw LZ4 **block**-format compressed chunks, concatenated back-to-back |

### Key insight: chunks are *not* separate images

The first (and most fragile) assumption to get right: the N chunks are
**not** N separate sprites, frames, or UI elements. They're just fixed-size
(256 KB decompressed) pieces of **one single decompressed byte stream**,
split up purely so each chunk stays under a manageable block-compression
size. Once every chunk is decompressed and the results are concatenated in
order, the result is one continuous buffer — which happens to be a complete,
standard, uncompressed BMP file (magic `BM`, standard `BITMAPFILEHEADER` +
`BITMAPINFOHEADER`, 24 bits per pixel).

Confirmation this reading is correct:

- `total decompressed size` (`0x278D38` = 2,592,056) is essentially
  `width × height × 3` (1920 × 450 × 3 = 2,592,000) plus a 54-byte BMP header
  — exactly what you'd expect from one raw 24bpp bitmap.
- `total compressed size` field (42,868) exactly equals the header size
  (`0x48` = 72 bytes) **plus** the sum of the 10 chunk sizes in the size
  table (42,796) — a strong internal consistency check that the header
  layout is being parsed correctly.
- The last chunk decompresses to a *smaller* amount than the others
  (232,760 bytes vs. 262,144) — exactly the remainder needed to reach the
  total decompressed size, consistent with fixed-size chunking of one
  stream rather than N independently-sized images.

### Why the container isn't standard LZ4

Standard LZ4 comes in two relevant flavors:

- **LZ4 frame format** — starts with magic `0x184D2204`, includes block
  headers/checksums, decoded by the high-level `LZ4_decompress` / "frame"
  APIs.
- **LZ4 block format** — the raw compressed byte stream with *no* magic, no
  length prefix, no checksums. The decoder must be told the output size
  separately, because nothing in the data encodes it.

Each chunk in an LZ4P8 file is **raw block format**, with the output size
supplied externally by the chunk-size table's implied "256 KB per chunk
except possibly the last" logic (offset `0x0C` × `0x10`, capped by the total
at `0x04`). Feeding this data into a frame-format decoder (e.g. calling
`lz4.decode()` from most JS LZ4 libraries) fails silently or throws, because
it's looking for a frame magic that doesn't exist.

---

## 3. BMP payload

Once concatenated, the decompressed data is a plain-vanilla Windows BMP:

| Offset (in decompressed stream) | Size | Field |
|---|---|---|
| `0x00` | 2 bytes | `"BM"` signature |
| `0x02` | u32 | File size |
| `0x06` | 4 bytes | Reserved |
| `0x0A` | u32 | Pixel data offset (`0x36` = 54 in the sample) |
| `0x0E` | u32 | DIB header size (`0x28` = 40 → `BITMAPINFOHEADER`) |
| `0x12` | i32 | Width in pixels |
| `0x16` | i32 | Height in pixels (positive = bottom-up row order) |
| `0x1A` | u16 | Color planes (always 1) |
| `0x1C` | u16 | Bits per pixel (`24` in the sample) |

Sample file: **1920 × 450**, 24 bits per pixel, no alpha channel, no
compression, rows stored bottom-to-top and padded to 4-byte boundaries
(standard BMP row alignment) — pixel channel order is **BGR**, not RGB.

---

## 4. Extraction pipeline

```
LZ4P8 file
   │
   ├─ read header (sizes, chunk count, chunk size table)
   │
   ├─ for each chunk:
   │     decompress raw LZ4 block → fixed-size output buffer
   │     (256 KB, except the final chunk = remainder)
   │
   ├─ concatenate all decompressed chunks in order
   │     → one continuous byte buffer
   │
   └─ that buffer IS a complete, valid BMP file
        (magic "BM" at byte 0, standard header, raw BGR pixels)
```

### LZ4 block decompression (algorithm)

The tool implements a minimal, dependency-free raw-block LZ4 decompressor
(~25 lines, no libraries). LZ4 block format is a simple token-based scheme:

1. Read a **token** byte. High nibble = literal length, low nibble = match
   length (both `0–15`, with `15` meaning "read more length bytes that
   follow, each `0xFF` adding 255").
2. Copy `literal length` bytes directly from the compressed stream to the
   output — these are literal, unmatched bytes.
3. If more data follows: read a 2-byte little-endian **offset**, telling you
   how far back in the *output* buffer the next match starts.
4. Copy `match length + 4` bytes from `output[position - offset]` forward
   (byte-by-byte, since matches can overlap the copy destination —
   important for run-length-style repeats like solid color fills).
5. Repeat until the chunk's input is exhausted.

This is the same logic used by every LZ4 block decoder (including
`lz4js`'s `decompressBlock`); it was verified byte-for-byte identical
against both `lz4js` (Node) and Python's `lz4.block` module on the sample
file before being inlined into the HTML tool.

---

## 5. Using the tool

1. Open `lz4bm` in any modern browser (no server needed).
2. Drop your `LZ4P8`-tagged file onto the drop zone, or click to browse.
3. The tool will:
   - Validate the magic bytes.
   - Decompress every chunk and log per-chunk compressed/decompressed
     sizes.
   - Reassemble the BMP and render it live on a `<canvas>` (manual pixel
     decode is used instead of `<img src="blob:...">`, since BMP rendering
     support via `<img>` varies across embedding contexts / sandboxed
     iframes).
   - Show a **Technical details** panel: source device/partition (parsed
     from filenames like `mmcblk0p25`), resolution, color format, header and
     pixel-data offsets, compression ratio, and a full chunk offset/size
     table.
   - Offer a **Download BMP** button with the exact reassembled bytes.

### Filename convention for auto-detection

If your file is named like a raw partition dump (`mmcblkNpM`, e.g.
`mmcblk0p25`), the tool automatically identifies it as MMC block `N`,
partition `M`. Rename accordingly if you want that card populated.

---

## 6. Open questions / things to check on other samples

- **Bytes `0x14`–`0x1F`** are unidentified (zeroed in this sample). Worth
  diffing against other LZ4P8 files from the same device/firmware to see if
  they ever vary — could be a checksum, format version, or flags field.
- Whether `bpp` is ever anything other than 24 on this device family. The
  tool currently only renders the canvas preview for 24bpp; other depths
  will still decompress and download correctly, but won't preview.
- Whether chunk size is always exactly 256 KB (`0x40000`) across firmware
  versions, or if it's tunable — currently read from the header rather than
  hardcoded, so this should be robust either way.
