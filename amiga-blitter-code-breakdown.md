# Amiga Blitter Code Line-by-Line Analysis (main.c:517–527) – Cookie-Cut Masked Sprite Blit

## Overview

The following code block from `main.c` (lines 517‑527) configures the Amiga's blitter hardware to perform a **masked sprite blit** (often called a "cookie‑cut" operation). It draws a 64×64‑pixel, 5‑bitplane BOB (Blitter Object) with a separate 1‑bit mask onto a 320×256 background image. The sprite is placed at screen coordinates `(x, y)` with pixel‑accurate horizontal positioning via the barrel shifter.

```c
WaitBlit();
custom->bltcon0 = 0xca | SRCA | SRCB | SRCC | DEST | ((x & 15) << ASHIFTSHIFT);
custom->bltcon1 = ((x & 15) << BSHIFTSHIFT);
custom->bltapt = src;
custom->bltamod = 64 / 8;           // 8 bytes: skip mask to next plane's image data
custom->bltbpt = src + 64 / 8 * 1; // mask is 8 bytes after image data
custom->bltbmod = 64 / 8;           // 8 bytes: skip image to next plane's mask data
custom->bltcpt = custom->bltdpt = (UBYTE *)image + 320 / 8 * 5 * y + x / 8;
custom->bltcmod = custom->bltdmod = (320 - 64) / 8; // 32 bytes
custom->bltafwm = custom->bltalwm = 0xffff;
custom->bltsize = ((64 * 5) << HSIZEBITS) | (64 / 16);
```

## Hardware Register Reference

| Register  | Address    | Purpose                                                                 |
| --------- | ---------- | ----------------------------------------------------------------------- |
| `BLTCON0` | `0xDFF040` | Blitter control 0 – minterm, channel enables, shift direction           |
| `BLTCON1` | `0xDFF042` | Blitter control 1 – additional shift, fill mode, line mode              |
| `BLTAPT`  | `0xDFF050` | Pointer to channel A (source image) data                                |
| `BLTAMOD` | `0xDFF052` | Modulo for channel A (bytes to skip after each line)                    |
| `BLTBPT`  | `0xDFF054` | Pointer to channel B (mask) data                                        |
| `BLTBMOD` | `0xDFF056` | Modulo for channel B                                                    |
| `BLTCPT`  | `0xDFF058` | Pointer to channel C (background) data                                  |
| `BLTCMOD` | `0xDFF05A` | Modulo for channel C                                                    |
| `BLTDPT`  | `0xDFF05C` | Pointer to channel D (destination) data                                 |
| `BLTDMOD` | `0xDFF05E` | Modulo for channel D                                                    |
| `BLTAFWM` | `0xDFF044` | First‑word mask – which bits of the first word are used                 |
| `BLTALWM` | `0xDFF046` | Last‑word mask – which bits of the last word are used                   |
| `BLTSIZE` | `0xDFF058` | Blitter size – starts the blit; upper 10 bits = height, lower 6 = width |

## Line‑by‑Line Breakdown

### Line 517: `WaitBlit();`

**What it does** – Calls an inline function that polls the `DMACONR` register bit 14 (BLTBUSY) and loops until the blitter is idle.

**Why** – The Amiga blitter can only process one operation at a time; writing new parameters while a blit is in progress corrupts the operation.

### Line 518: `custom->bltcon0 = 0xca | SRCA | SRCB | SRCC | DEST | ((x & 15) << ASHIFTSHIFT);`

**Register**: `BLTCON0` – Blitter Control 0

**Bit breakdown**:

| Bits  | Value            | Meaning                                                                        |
| ----- | ---------------- | ------------------------------------------------------------------------------ |
| 15‑12 | `(x & 15) << 12` | Channel A barrel‑shift amount (0‑15 pixels right)                              |
| 11‑8  | `0x0c`           | Channel B shift amount (not used here; set via `BLTCON1`)                      |
| 7‑0   | `0xca`           | **LF (Logic Function) minterm** – defines the boolean combination of A,B,C → D |
| –     | `SRCA` (0x0200)  | Enable DMA for channel A (source image)                                        |
| –     | `SRCB` (0x0400)  | Enable DMA for channel B (mask)                                                |
| –     | `SRCC` (0x0800)  | Enable DMA for channel C (background)                                          |
| –     | `DEST` (0x1000)  | Enable DMA for channel D (destination)                                         |

**Minterm `0xca`** (binary `11001010`):

- Implements the boolean expression `(A & B & C) | (~A & C)`.
- In cookie‑cut terms:
  - Where the mask (B) is **1** → output = source pixel (A).
  - Where the mask (B) is **0** → output = background pixel (C).
- This yields a transparent sprite where the mask defines the visible shape.

**Shift amount `(x & 15)`**

- The low 4 bits of the X coordinate determine the pixel‑offset within a 16‑pixel (1‑word) boundary.
- Because the blitter works on word‑aligned memory, drawing at a non‑word‑aligned X requires shifting the source data right by 0‑15 pixels.
- The shift is applied to channel A (image) and must be matched by an identical shift for channel B (mask); see `BLTCON1`.

**Macro values** (typical Amiga definitions):

- `SRCA` = `0x0200` (bit 9)
- `SRCB` = `0x0400` (bit 10)
- `SRCC` = `0x0800` (bit 11)
- `DEST` = `0x1000` (bit 12)
- `ASHIFTSHIFT` = `12` (bits 15‑12 for channel A shift)

### Line 519: `custom->bltcon1 = ((x & 15) << BSHIFTSHIFT);`

**Register**: `BLTCON1` – Blitter Control 1

**Bit breakdown**:

| Bits  | Value            | Meaning                                                        |
| ----- | ---------------- | -------------------------------------------------------------- |
| 15‑12 | `(x & 15) << 12` | Channel B barrel‑shift amount (must match channel A shift)     |
| 11‑0  | `0`              | Unused (line‑mode, fill, and extra‑half‑bright flags are zero) |

**Why match the shift** – The mask (channel B) must be shifted by the same amount as the image (channel A) so that each mask bit stays aligned with its corresponding source pixel. If the shifts differ, the transparency would be misaligned, causing visual artifacts.

**Macro**: `BSHIFTSHIFT` = `12` (bits 15‑12 for channel B shift)

### Line 520: `custom->bltapt = src;`

**Register**: `BLTAPT` – Blitter A Pointer

**Value**: `src` points to the start of the sprite's image data (plane 0, line 0).

**Memory layout** – The sprite data (`bob2`) is stored in **interleaved mask format**: each bitplane's 64‑pixel line (8 bytes) is immediately followed by its 1‑bit mask (another 8 bytes). Thus the data for plane 0 looks like:

`[Plane0 line0 (8 bytes)][Plane0 mask0 (8 bytes)][Plane0 line1][Plane0 mask1]...`

After all lines of plane 0, the same pattern repeats for plane 1, etc.

### Line 521: `custom->bltamod = 64 / 8;`

**Register**: `BLTAMOD` – Blitter A Modulo

**Value**: `8` (64 pixels ÷ 8 bits/byte = 8 bytes per line).

**Why** – After the blitter reads one line (8 bytes) of image data, it must skip over the accompanying mask (8 bytes) to reach the next line's image data. Adding `BLTAMOD` to the pointer after each line accomplishes this skip.

### Line 522: `custom->bltbpt = src + 64 / 8 * 1;`

**Register**: `BLTBPT` – Blitter B Pointer

**Value**: `src + 8` (points 8 bytes ahead of `src`).

**Why** – The mask for the first line resides 8 bytes after the start of the image data (see memory layout above). This register tells the blitter where the 1‑bit mask begins.

### Line 523: `custom->bltbmod = 64 / 8;`

**Register**: `BLTBMOD` – Blitter B Modulo

**Value**: `8` (same as `BLTAMOD`).

**Why** – After reading a mask line (8 bytes), the pointer must skip the following image line (8 bytes) to reach the next mask line. This modulo keeps the mask pointer synchronized with the image pointer across the interleaved data.

### Line 524: `custom->bltcpt = custom->bltdpt = (UBYTE *)image + 320 / 8 * 5 * y + x / 8;`

**Registers**: `BLTCPT` (background read pointer) and `BLTDPT` (destination write pointer)

**Value**: address in the background bitmap where the sprite will be drawn.

**Calculation breakdown**:

- `320 / 8` = 40 bytes per scanline (320 pixels, 8 bits/byte).
- Multiply by `5` because the background uses **5 interleaved bitplanes** – each plane's line is stored consecutively (planar, not interleaved with mask).
- `320 / 8 * 5 * y` = byte offset to the start of line `y` in the bitmap.
- `x / 8` = byte offset within that line (word‑aligned; the low 3 bits of `x` are handled by the barrel shifter).

**Why both pointers are equal** – In a cookie‑cut operation, the blitter reads the background from channel C and writes the result back to the same location via channel D. Setting both pointers to the same address ensures the blitter reads the original background before overwriting it.

### Line 525: `custom->bltcmod = custom->bltdmod = (320 - 64) / 8;`

**Registers**: `BLTCMOD` and `BLTDMOD` – Modulo for channels C and D

**Value**: `32` ( (320 − 64) ÷ 8 ).

**Why** – After blitting a 64‑pixel (8‑byte) wide line, the pointer must advance to the start of the next line in the destination bitmap. Because the destination is 320 pixels (40 bytes) wide, skipping `40‑8 = 32` bytes moves the pointer from the end of the blitted region to the beginning of the next line.

**Note**: The modulo is applied **after** each line, so the pointer effectively jumps over the "unused" part of the screen line.

### Line 526: `custom->bltafwm = custom->bltalwm = 0xffff;`

**Registers**: `BLTAFWM` (First‑Word Mask) and `BLTALWM` (Last‑Word Mask)

**Value**: `0xffff` (all 16 bits enabled).

**Why** – When blitting a region that is not a whole number of words, the masks define which bits of the first and last word are actually used. Here the sprite width is exactly 64 pixels = 4 words (64÷16), so every word is fully inside the blit rectangle. Setting both masks to `0xffff` ensures all bits are transferred.

### Line 527: `custom->bltsize = ((64 * 5) << HSIZEBITS) | (64 / 16);`

**Register**: `BLTSIZE` – Blitter Size (starts the blit)

**Bit breakdown**:

| Bits | Value         | Meaning                                                    |
| ---- | ------------- | ---------------------------------------------------------- |
| 15‑6 | `(64*5) << 6` | **Height** = 320 blit‑lines (64 pixel‑lines × 5 bitplanes) |
| 5‑0  | `64 / 16` = 4 | **Width** = 4 words (64 pixels ÷ 16 pixels/word)           |

**Why height = 64×5** – The blitter processes **all five bitplanes in a single operation**. Each "blit‑line" corresponds to one source‑line of one plane. Because the data is interleaved (image+mask per plane), the blitter automatically advances through planes via the modulo settings. Thus 64 pixel‑lines × 5 planes = 320 blit‑lines.

**Why width = 4 words** – The sprite is 64 pixels wide; at 16 pixels per word (low‑resolution), that's exactly 4 words.

**Macro**: `HSIZEBITS` = `6` (the height value is placed in bits 15‑6).

**Important**: Writing to `BLTSIZE` **immediately starts the blit**. The CPU can continue executing while the blitter works (though here we wait for completion before the next iteration).

## Summary of the Operation

1. **Wait** – Ensure previous blit is finished.
2. **Configure logic** – Set minterm `0xca` for cookie‑cut transparency, enable all four DMA channels (A=image, B=mask, C=background, D=destination).
3. **Set pixel‑alignment** – Barrel‑shift both image and mask by `(x & 15)` pixels.
4. **Point to data** – Image pointer (`BLTAPT`) at start of sprite, mask pointer (`BLTBPT`) 8 bytes later.
5. **Set modulos** – Skip 8 bytes after each line to navigate the interleaved image+mask layout.
6. **Point to destination** – Background/destination pointer at screen coordinate `(x, y)` in the 5‑plane bitmap.
7. **Set destination modulo** – Skip 32 bytes after each line to move to the next screen line.
8. **Disable edge masking** – Use full words (`0xffff`).
9. **Start the blit** – Write height (320) and width (4) to `BLTSIZE`.

The result is a 64×64 sprite drawn with perfect transparency at the specified `(x, y)` position, with pixel‑accurate horizontal placement thanks to the barrel shifter.
