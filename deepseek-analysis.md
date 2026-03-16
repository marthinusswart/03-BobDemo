## Detailed Analysis of Amiga Blitter BOB Drawing Code

This code performs a **masked sprite blit** (Blitter Object) operation, drawing a 64x64 pixel, 5-bitplane BOB with transparency onto a background image at position (x, y). Here's the line-by-line breakdown:

### Line 1: [`WaitBlit()`](main.c:515)

```c
WaitBlit();
```

- Waits for any previous blitter operation to complete by polling DMACONR bit 14
- Essential to prevent corrupting an ongoing blit operation
- The blitter can only handle one operation at a time

### Line 2: [`custom->bltcon0`](main.c:516)

```c
custom->bltcon0 = 0xca | SRCA | SRCB | SRCC | DEST | ((x & 15) << ASHIFTSHIFT);
```

- **BLTCON0**: Blitter Control Register 0 (minterm and channel configuration)
- **`0xca`**: LF (Logic Function) minterm = `(A & B & C) | (~A & C)` = Cookie-cut operation
  - Where A=1 and B=1 (mask allows): use source pixel
  - Where mask doesn't allow: preserve background (C)
- **`SRCA | SRCB | SRCC | DEST`**: Enables all 4 DMA channels (A=image, B=mask, C=background, D=destination)
- **`((x & 15) << ASHIFTSHIFT)`**: Sets channel A's barrel shifter (0-15 pixels)
  - Handles pixel-aligned blitting when x position isn't word-aligned
  - ASHIFTSHIFT = 12, so shift amount goes in bits 15-12

### Line 3: [`custom->bltcon1`](main.c:517)

```c
custom->bltcon1 = ((x & 15) << BSHIFTSHIFT);
```

- **BLTCON1**: Blitter Control Register 1 (additional shift control)
- **`((x & 15) << BSHIFTSHIFT)`**: Sets channel B's barrel shifter to match A's shift
  - BSHIFTSHIFT = 12, so shift amount goes in bits 15-12
  - Both A (image) and B (mask) must shift together to maintain alignment
  - Bits 0-15 control the shift amount and direction

### Line 4: [`custom->bltapt`](main.c:518)

```c
custom->bltapt = src;
```

- **BLTAPT**: Blitter A Pointer - source data pointer
- Points to the start of the BOB image data (without mask)
- Source is [`bob2`](main.c:513) which is a 64x64 pixel, 5-bitplane sprite

### Line 5: [`custom->bltamod`](main.c:519)

```c
custom->bltamod = 64 / 8;  // 8 bytes: skip mask to next plane's image data
```

- **BLTAMOD**: Blitter A Modulo (8 bytes)
- After each line, advance pointer by 8 bytes to skip the mask data
- Memory layout: [Plane0 image][Plane0 mask][Plane1 image][Plane1 mask]...
- Each scanline is 64 pixels = 8 bytes, mask is interleaved, so skip 8 bytes to reach next plane

### Line 6: [`custom->bltbpt`](main.c:520)

```c
custom->bltbpt = src + 64 / 8 * 1;  // mask is 8 bytes after image data
```

- **BLTBPT**: Blitter B Pointer - mask data pointer
- Offset by 8 bytes (64 pixels ÷ 8 bits) from source to point at mask data
- Mask controls which pixels are drawn (1=draw, 0=transparent)

### Line 7: [`custom->bltbmod`](main.c:521)

```c
custom->bltbmod = 64 / 8;  // 8 bytes: skip image to next plane's mask data
```

- **BLTBMOD**: Blitter B Modulo (8 bytes)
- Same as BLTAMOD - skips 8 bytes to reach the next plane's mask
- Keeps mask and image data synchronized across bitplanes

### Line 8: [`custom->bltcpt`](main.c:522) and [`custom->bltdpt`](main.c:522)

```c
custom->bltcpt = custom->bltdpt = (UBYTE *)image + 320 / 8 * 5 * y + x / 8;
```

- **BLTCPT**: Blitter C Pointer - background read pointer
- **BLTDPT**: Blitter D Pointer - destination write pointer
- Both point to the same location in the background image
- **`320 / 8 * 5 * y`**: Calculate Y offset (40 bytes/line × 5 planes × y)
- **`+ x / 8`**: Add X offset in bytes (word-aligned position)
- Screen width is 320 pixels = 40 bytes per line, 5 interleaved planes

### Line 9: [`custom->bltcmod`](main.c:523) and [`custom->bltdmod`](main.c:523)

```c
custom->bltcmod = custom->bltdmod = (320 - 64) / 8;  // 32 bytes
```

- **BLTCMOD/BLTDMOD**: Blitter C and D Modulo (32 bytes)
- After blitting each 64-pixel (8-byte) line, skip 32 bytes to reach the next line
- Calculation: (screen_width - bob_width) / 8 = (320 - 64) / 8 = 32 bytes
- This advances the pointer to the correct Y position in the interleaved bitplane format

### Line 10: [`custom->bltafwm`](main.c:524) and [`custom->bltalwm`](main.c:524)

```c
custom->bltafwm = custom->bltalwm = 0xffff;
```

- **BLTAFWM**: Blitter First Word Mask (0xffff = all bits enabled)
- **BLTALWM**: Blitter Last Word Mask (0xffff = all bits enabled)
- `0xffff` means no edge masking - use all bits in first and last words
- Used to clip edges when blitting partial words

### Line 11: [`custom->bltsize`](main.c:525)

```c
custom->bltsize = ((64 * 5) << HSIZEBITS) | (64 / 16);  // 64 lines * 5 planes, 4 words wide
```

- **BLTSIZE**: Blitter Size - starts the blit operation immediately upon write
- **`(64 * 5) << HSIZEBITS`**: Height = 320 lines (64 pixel-lines × 5 bitplanes)
  - HSIZEBITS = 6, so value goes in bits 15-6
  - All planes are blitted consecutively in one operation
- **`(64 / 16)`**: Width = 4 words (64 pixels ÷ 16 pixels/word)
  - Value goes in bits 5-0
  - Maximum width is 64 words (1024 pixels)

## Summary

This is a **cookie-cut masked blit** that:

1. Uses minterm `0xca` to blend source through mask onto background
2. Handles pixel-perfect horizontal positioning via barrel shifters
3. Processes 5 interleaved bitplanes with masks in a single operation
4. Draws a 64×64 pixel BOB at position (x, y) on a 320×256 screen
5. Uses the mask (channel B) to provide transparency

The interleaved data format stores: **[Plane0_line][Plane0_mask][Plane1_line][Plane1_mask]...** which is why the modulos skip 8 bytes between planes.
