# Amiga Blitter Cookie-Cut Blit Analysis

## Overview

This document provides a detailed line-by-line analysis of an Amiga blitter operation that performs a **masked cookie-cut blit**. This technique draws a 64×64 pixel, 5-bitplane sprite (bob) with transparency onto a background image.

## Source Code

```c
WaitBlit();
custom->bltcon0 = 0xca | SRCA | SRCB | SRCC | DEST | ((x & 15) << ASHIFTSHIFT);
custom->bltcon1 = ((x & 15) << BSHIFTSHIFT);
custom->bltapt = src;
custom->bltamod = 64 / 8;           // 8 bytes: skip mask to next plane's image data
custom->bltbpt = src + 64 / 8 * 1;  // mask is 8 bytes after image data
custom->bltbmod = 64 / 8;           // 8 bytes: skip image to next plane's mask data
custom->bltcpt = custom->bltdpt = (UBYTE *)image + 320 / 8 * 5 * y + x / 8;
custom->bltcmod = custom->bltdmod = (320 - 64) / 8; // 32 bytes
custom->bltafwm = custom->bltalwm = 0xffff;
custom->bltsize = ((64 * 5) << HSIZEBITS) | (64 / 16); // 64 lines * 5 planes, 4 words wide
```

**Context from [`main.c`](main.c:516-525):**

- Sprite dimensions: 64×64 pixels
- Bitplanes: 5 (32 colors)
- Background width: 320 pixels
- Bob data format: Interleaved image and mask data per bitplane

---

## Line-by-Line Detailed Analysis

### Line 1: Wait for Blitter Availability

```c
WaitBlit();
```

**Purpose:** Ensures the blitter hardware is idle before configuring a new operation.

**Details:**

- The Amiga blitter can only process one operation at a time
- Attempts to configure it while busy will cause data corruption
- Implementation in [`main.c:83-90`](main.c:83) polls the DMACONR register bit 14
- Includes A1000 compatibility read to avoid hardware bugs

---

### Line 2: BLTCON0 - Blitter Control Register 0

```c
custom->bltcon0 = 0xca | SRCA | SRCB | SRCC | DEST | ((x & 15) << ASHIFTSHIFT);
```

**Purpose:** Configures the blitter's logic function, active channels, and channel A barrel shifter.

#### Component Breakdown:

**`0xca` - Logic Function (Minterm)**

- Binary: `11001010`
- Boolean formula: `D = (A & B) | (C & ~B)`
- Meaning: "Use A (source) where B (mask) is 1, else use C (background)"
- This is the classic **cookie-cut** operation for transparency

**Minterm Truth Table:**
| C | B | A | Result |
|---|---|---|--------|
| 0 | 0 | 0 | 0 |
| 0 | 0 | 1 | 0 |
| 0 | 1 | 0 | 0 |
| 0 | 1 | 1 | 1 |
| 1 | 0 | 0 | 1 |
| 1 | 0 | 1 | 1 |
| 1 | 1 | 0 | 0 |
| 1 | 1 | 1 | 1 |

Reading bottom-to-top: `11001010` = `0xCA`

**`SRCA | SRCB | SRCC | DEST`**

- Enables all four DMA channels:
  - **SRCA**: Source image data
  - **SRCB**: Mask data
  - **SRCC**: Background/destination read
  - **DEST**: Destination write

**`((x & 15) << ASHIFTSHIFT)`**

- Configures the barrel shifter for channel A
- `x & 15`: Extracts the pixel offset within a 16-pixel word (0-15)
- Allows blitting at any pixel position, not just word boundaries
- The shifter rotates the source data right by this amount

**Example:** If x=10, the source is shifted right by 10 pixels to align with the destination

---

### Line 3: BLTCON1 - Blitter Control Register 1

```c
custom->bltcon1 = ((x & 15) << BSHIFTSHIFT);
```

**Purpose:** Configures the barrel shifter for channel B (mask).

**Details:**

- Must match the shift value from BLTCON0
- Ensures the mask aligns perfectly with the shifted source image
- Without this, the mask would not correspond to the correct source pixels
- Also controls descending mode (not used here - ascending mode)

**Why Both A and B Need Shifting:**

- Channel A contains the sprite image data
- Channel B contains the mask
- Both must shift together to maintain their spatial relationship
- If only A shifted, the mask would apply to the wrong pixels

---

### Line 4: BLTAPT - Channel A Pointer (Source Image)

```c
custom->bltapt = src;
```

**Purpose:** Points to the start of the source sprite image data.

**Details:**

- `src` points to [`bob2`](main.c:169) (the 64×64 sprite data)
- Contains interleaved bitplane image data
- Format: [plane0_image][plane0_mask][plane1_image][plane1_mask]...
- Each scanline of one bitplane is 64 pixels ÷ 8 = 8 bytes

---

### Line 5: BLTAMOD - Channel A Modulo

```c
custom->bltamod = 64 / 8;  // 8 bytes: skip mask to next plane's image data
```

**Purpose:** Defines how many bytes to skip after each scanline of channel A.

**Details:**

- **Value:** 8 bytes
- **Why:** Bob data is organized as [image_line][mask_line] for each bitplane
- After reading 8 bytes of image data, we need to skip 8 bytes of mask data
- This brings us to the next bitplane's image data

**Memory Layout Visualization:**

```
[Plane 0 Image Line 0: 8 bytes]
[Plane 0 Mask Line 0:  8 bytes] <- skip this
[Plane 1 Image Line 0: 8 bytes]
[Plane 1 Mask Line 0:  8 bytes] <- skip this
[Plane 2 Image Line 0: 8 bytes]
...
```

**Modulo Calculation:**

- Read width: 8 bytes
- Skip: 8 bytes (the mask)
- Total advance: 16 bytes per iteration
- Modulo = 8 bytes (added after the read)

---

### Line 6: BLTBPT - Channel B Pointer (Mask)

```c
custom->bltbpt = src + 64 / 8 * 1;  // mask is 8 bytes after image data
```

**Purpose:** Points to the start of the mask data.

**Details:**

- **Offset:** `64 / 8 * 1` = 8 bytes from the source pointer
- The mask for each bitplane immediately follows its image data
- Mask format: 1 bit per pixel (1 = opaque, 0 = transparent)
- Each mask scanline is also 8 bytes (64 pixels ÷ 8)

**Why +8 bytes:**

- First 8 bytes = Plane 0 image data
- Next 8 bytes = Plane 0 mask data (we point here)

---

### Line 7: BLTBMOD - Channel B Modulo

```c
custom->bltbmod = 64 / 8;  // 8 bytes: skip image to next plane's mask data
```

**Purpose:** Defines how many bytes to skip after each scanline of channel B.

**Details:**

- **Value:** 8 bytes
- **Why:** After reading a mask line, skip the next bitplane's image data
- Brings us to the next bitplane's mask data

**Memory Layout for Mask Reading:**

```
[Plane 0 Mask Line 0:  8 bytes] <- read this
[Plane 1 Image Line 0: 8 bytes] <- skip this
[Plane 1 Mask Line 0:  8 bytes] <- read this
[Plane 2 Image Line 0: 8 bytes] <- skip this
[Plane 2 Mask Line 0:  8 bytes] <- read this
...
```

---

### Line 8: BLTCPT and BLTDPT - Channel C and D Pointers

```c
custom->bltcpt = custom->bltdpt = (UBYTE *)image + 320 / 8 * 5 * y + x / 8;
```

**Purpose:** Sets both background read (C) and destination write (D) pointers to the same location.

**Details:**

- **Both point to the same address** in the destination bitmap
- Channel C reads the existing background
- Channel D writes the composited result
- This is typical for cookie-cut operations

**Address Calculation Breakdown:**

**`(UBYTE *)image`**

- Base address of the 320×256, 5-bitplane background image

**`320 / 8 * 5 * y`**

- Vertical offset to scanline `y`
- `320 / 8` = 40 bytes per scanline
- `* 5` = 5 bitplanes (interleaved)
- `* y` = jump down y scanlines

**`x / 8`**

- Horizontal offset to byte containing pixel x
- Integer division aligns to word boundary
- Sub-pixel offset handled by barrel shifter

**Example:** For position (x=96, y=200):

- Vertical: `40 * 5 * 200 = 40,000` bytes
- Horizontal: `96 / 8 = 12` bytes
- Total offset: `40,012` bytes from image start

---

### Line 9: BLTCMOD and BLTDMOD - Channel C and D Modulos

```c
custom->bltcmod = custom->bltdmod = (320 - 64) / 8;  // 32 bytes
```

**Purpose:** Defines how many bytes to skip after each scanline for channels C and D.

**Details:**

- **Value:** `(320 - 64) / 8 = 32 bytes`
- **Calculation:**
  - Screen width: 320 pixels = 40 bytes
  - Blit width: 64 pixels = 8 bytes
  - After writing 8 bytes, skip 32 bytes to reach the next scanline

**Why This Works:**

- Each scanline we read/write 8 bytes (64 pixels)
- The screen is 40 bytes wide
- To move to the next line at the same x position: skip (40 - 8) = 32 bytes

**Visual Representation:**

```
Line 0: [read/write 8 bytes]...[skip 32 bytes]
Line 1: [read/write 8 bytes]...[skip 32 bytes]
Line 2: [read/write 8 bytes]...[skip 32 bytes]
...
```

---

### Line 10: BLTAFWM and BLTALWM - First/Last Word Masks

```c
custom->bltafwm = custom->bltalwm = 0xffff;
```

**Purpose:** Masks the first and last words of each blit line.

**Details:**

- **BLTAFWM:** First word mask - controls which bits are affected in the leftmost word
- **BLTALWM:** Last word mask - controls which bits are affected in the rightmost word
- **Value `0xffff`:** All bits enabled (no masking)

**Why All Bits Enabled:**

- This 64-pixel wide blit is 4 words (64 ÷ 16 = 4)
- We want to process all pixels in every word
- No edge clipping needed

**When These Matter:**

- If blitting partial words at edges (e.g., 17 pixels = 2 words, but only 1 bit in second word)
- Edge clipping for screen boundaries
- Irregular shaped sprites

---

### Line 11: BLTSIZE - Trigger the Blit Operation

```c
custom->bltsize = ((64 * 5) << HSIZEBITS) | (64 / 16);  // 64 lines * 5 planes, 4 words wide
```

**Purpose:** Sets the blit dimensions and **immediately starts the blitter**.

**Critical:** Writing to BLTSIZE triggers the operation. All other registers must be configured first.

**Component Breakdown:**

**`(64 * 5) << HSIZEBITS`**

- Height value: 64 scanlines × 5 bitplanes = **320 blit lines**
- Shifted to upper bits (HSIZEBITS = 6, typically)
- The blitter processes all bitplanes as one tall operation

**`(64 / 16)`**

- Width value: 64 pixels ÷ 16 pixels per word = **4 words**
- Goes in the lower 6 bits
- Maximum width: 64 words (1024 pixels)

**Combined Value:**

- Height: 320 (bits 15-6)
- Width: 4 (bits 5-0)
- Binary: `0001010000000100` = 0x1404

**Why 64 × 5 = 320 lines?**

- The blitter doesn't understand "bitplanes"
- It just processes data linearly with the configured modulos
- We blit 5 complete bitplanes back-to-back as one tall operation
- The modulos handle jumping between interleaved data

---

## Summary

### Operation Overview

This code performs a **cookie-cut blit** operation that:

1. Draws a 64×64 pixel sprite at position (x, y)
2. Uses a mask for transparency
3. Preserves the background where the mask is 0
4. Handles pixel-perfect positioning using barrel shifters
5. Works with 5 bitplanes (32 colors)

### Key Techniques

**Three-Channel Cookie-Cut:**

- Channel A: Source sprite image
- Channel B: Transparency mask
- Channel C: Background (read)
- Channel D: Destination (write, same as C)
- Minterm 0xCA: Combine using mask

**Barrel Shifters:**

- Handle non-word-aligned x positions (0-15 pixel offsets)
- Both A and B channels shift together to maintain alignment

**Modulo Magic:**

- Channel A/B modulos: Navigate interleaved image/mask data
- Channel C/D modulos: Skip to next scanline in destination bitmap

**Interleaved Data Format:**

- Bob data: [plane0_img][plane0_mask][plane1_img][plane1_mask]...
- Screen data: [line_plane0][line_plane1]...[line_plane4][next_line_plane0]...

### Performance Notes

- Total memory transfers: (4 words × 320 lines × 4 channels) = 10,240 words
- At 1 word per DMA cycle: ~10,240 cycles minimum
- Actual time depends on CPU/blitter DMA contention
- Barrel shifting adds no additional cycles (handled by hardware)

### Common Pitfalls

1. **Forgetting WaitBlit()** - Can corrupt ongoing operations
2. **Mismatched shift values** - Causes mask misalignment
3. **Wrong modulos** - Data reads from incorrect memory locations
4. **Writing BLTSIZE first** - Blitter starts before configuration complete
5. **Incorrect minterm** - Wrong transparency or rendering artifacts

---

## References

- **Source File:** [`main.c`](main.c:516-525)
- **Sprite Data:** [`bob2`](main.c:169) - 64×64 pixels, 5 bitplanes with mask
- **Background:** [`image`](main.c:167) - 320×256 pixels, 5 bitplanes
- **Blitter Wait:** [`WaitBlit()`](main.c:83-90)

## Additional Resources

- Amiga Hardware Reference Manual - Blitter Chapter
- Hardware Constants: `SRCA`, `SRCB`, `SRCC`, `DEST`, `ASHIFTSHIFT`, `BSHIFTSHIFT`, `HSIZEBITS`
- Minterm Calculator: [https://wiki.amigaos.net/wiki/Blitter_Minterms](https://wiki.amigaos.net/wiki/Blitter_Minterms)
