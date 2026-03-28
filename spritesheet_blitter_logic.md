# Sprite Sheet and Blitter Logic Analysis

This document explains the end-to-end process of taking a modern PNG sprite sheet, converting it for Amiga hardware, and rendering it using the custom blitter chip.

## 1. PNG to BPL (The Conversion Process)

**The Problem:** Modern image files like PNGs are "chunky"—meaning the color of a pixel is defined by a single chunk of data (e.g., a byte representing an RGB value or a palette index). The Amiga hardware does not understand chunky pixels. It uses "planar" graphics, where an image is split into several separate 1-bit layers (bitplanes). For a 32-color image, it takes 5 separate bitplanes stacked on top of each other to form the final colors.

**The Process:**
When you run a tool like `kingcon.exe` to convert a PNG into a `.bpl` (Bitplane) file, it performs several operations:
1. **Quantization/Palettization:** It ensures the image uses a strict color palette (e.g., 32 colors).
2. **Chunky-to-Planar Conversion:** It separates the image into 5 distinct bitplanes.
3. **Mask Generation:** If configured to output a BOB (Blitter Object), it looks for the "transparent" color in the PNG (usually palette Index 0). It automatically generates an extra 1-bit layer (the mask) where every pixel is `0` if it's transparent, and `1` if it's visible.
4. **Interleaving:** To make life easier for the blitter hardware, the tool typically "interleaves" the data. Instead of storing all of Plane 0, then all of Plane 1, it stores them line-by-line: `[Plane 0 Line 0][Mask Line 0][Plane 1 Line 0][Mask Line 0]...`

**The Result:** The output `.bpl` is a raw binary blob of Amiga-ready planar data with an embedded, perfectly aligned mask.

---

## 2. Mask Blit Logic

**The Problem:** Once the `.bpl` is in memory, how does the hardware know which bytes are the image and which bytes are the mask?

**The Solution:** The blitter doesn't inherently "know" anything; it simply does exactly what the code tells it to do via memory pointers and modulos.

In the cookie-cut operation, the Amiga blitter uses 4 DMA channels:
*   **Channel A (Source Image):** The CPU points `BLTAPT` exactly at the first byte of the `.bpl` data.
*   **Channel B (Mask):** The CPU points `BLTBPT` offset by exactly the width of one image line. For a 64x64 sprite (8 bytes wide), `BLTBPT` is set to `source_address + 8`.

To keep them from reading each other's data as they move down the image, we use the **Modulo** registers (`BLTAMOD` and `BLTBMOD`). 
Because the data is interleaved (`[Image 8 bytes][Mask 8 bytes]`), we tell both channels to skip 8 bytes after every read. 
*   Channel A reads 8 bytes of image, skips the 8 bytes of mask, and lands on the next image data.
*   Channel B reads 8 bytes of mask, skips the 8 bytes of image, and lands on the next mask data.

They stay perfectly synchronized through the entire interleaved file.

---

## 3. Multi-BOB Sheet (PNG Masking)

**The Problem:** If you have a single large PNG containing a grid of multiple 16x16 sprites, how does the conversion tool know how to mask each individual 16x16 sprite?

**The Solution:** The tool doesn't actually need to know about the 16x16 boundaries to generate the mask. 

The mask generation is purely color-based, not boundary-based. When `kingcon` parses the PNG, it treats the entire sheet as one giant image. It simply applies a rule:
*   `If pixel_color == Index_0 (Transparent) -> Mask_bit = 0`
*   `If pixel_color != Index_0 (Visible) -> Mask_bit = 1`

This creates a global mask for the entire sprite sheet. The boundaries of the individual 16x16 sprites are implicitly handled because the empty space between the sprites on the sheet is filled with the transparent color, which seamlessly becomes `0`s in the mask layer.

---

## 4. Blitter with a Sprite Sheet

**The Problem:** If the `.bpl` is one giant sprite sheet, how does the blitter extract just one 16x16 sprite and its specific mask?

**The Solution:** This depends heavily on how the conversion tool was configured, leading to two different approaches:

### Approach A: The "Slicer" Method (Standard for Amiga)
Amiga developers rarely keep a 2D sprite sheet in memory. Navigating a 2D sheet containing interleaved mask and image data is mathematically very difficult for the blitter, because the modulo registers can only hold a single skip value (they can't "skip the mask, and then skip the rest of the sheet's width").

Instead, `kingcon` is usually told the dimensions of the sprites (e.g., 16x16) during conversion. It "slices" the PNG grid and saves the `.bpl` as a **linear array of standalone sprites**.

**Implementing the Slicer Method with Kingcon:**
To use this method, you explicitly tell kingcon the width and height of the individual sprites within the larger sprite sheet using dimension flags (e.g., `-W=16 -H=16`).

**Example command:**
```bat
kingcon amiga-bob.png ..\amiga-bob -Interleaved -Format=5 -RawPalette -Mask -W=16 -H=16
```

*(Note: Depending on the specific version of kingcon you are using, the arguments might be formatted as `-Width=16 -Height=16` or `-Bobs=16x16`. Check `kingcon -h` for your specific toolchain.)*

**What this command does to the output:**
1. **Changes the Interleaving Boundary:** Instead of interleaving an entire row of the whole sheet (e.g., 256 pixels wide), it will interleave exactly 16 pixels of image, followed by 16 pixels of mask, moving down until that single 16x16 sprite is complete.
2. **Creates a Linear Array:** Once the first sprite is completely written to the `.bpl` file, it moves to the next 16x16 cell in the PNG and appends it directly after the first one.
3. **Simplifies C Code:** Because the file is now a contiguous array of perfectly packaged sprites, your C code doesn't need to do any complex 2D pointer math. You can just calculate `sprite_size_bytes` and multiply it by the `sprite_index` to point the blitter directly at the frame you want to draw.

Memory looks like this:
`[Sprite 0 Data...][Sprite 1 Data...][Sprite 2 Data...]`

**How the blitter finds it:**
The code just does simple math to point to the correct sprite block:
```c
int sprite_size_bytes = (16 / 8) * 16 * 5 * 2; // (width * height * planes * (image+mask))
UBYTE* target_sprite = sheet_base_address + (sprite_index * sprite_size_bytes);

custom->bltapt = target_sprite;
custom->bltbpt = target_sprite + (16 / 8); // Offset by 1 line width
```
From there, it's treated exactly like the 64x64 single BOB example.

### Approach B: The 2D Sub-image Method
If the `.bpl` was converted as one giant 2D background, extracting a sprite requires complex pointer math:
```c
// Offset = (Y * bytes_per_sheet_row) + (X / 8)
UBYTE* target = sheet_base + (sprite_y * sheet_width_bytes * planes) + (sprite_x / 8);
```
However, because of the interleaved mask format limitations mentioned above, you cannot easily blit all 5 bitplanes of a sub-sprite in a single hardware operation from a 2D sheet. You would have to blit it one bitplane at a time, calculating the pointer and modulo for each plane individually. This is why **Approach A** is the industry standard for Amiga games.
