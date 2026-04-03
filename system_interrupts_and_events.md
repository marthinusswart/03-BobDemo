# System Control, Interrupts, and Events

This document explains how the BobDemo takes control of the Amiga hardware and manages interrupts and timing events.

## Overview

Classic Amiga demos and games need direct hardware access for precise timing and control. This requires temporarily taking over the system from the OS, then restoring it when done.

## System Takeover: TakeSystem()

Located at [main.c:92-126](main.c#L92-L126), this function seizes complete control of the Amiga hardware.

### Backup Phase
```c
Forbid();                      // Prevent task switching
SystemADKCON = custom->adkconr;  // Save Audio/Disk control
SystemInts = custom->intenar;    // Save interrupt enable state
SystemDMA = custom->dmaconr;     // Save DMA channel state
ActiView = GfxBase->ActiView;   // Save current screen view
```

The function first saves all critical system state that will need to be restored later.

### Display Clearing Phase
```c
LoadView(0);      // Clear the display
WaitTOF();        // Wait for Top Of Frame
WaitTOF();        // Wait again for stability
WaitVbl();        // Wait for vertical blank
WaitVbl();        // Wait again
```

Clears the screen and synchronizes with the display refresh cycle.

### Hardware Takeover Phase
```c
OwnBlitter();                  // Claim exclusive blitter access
WaitBlit();                    // Wait for any pending blits
Disable();                     // Disable OS task switching
custom->intena = 0x7fff;       // Disable ALL interrupts
custom->intreq = 0x7fff;       // Clear pending interrupt requests
custom->dmacon = 0x7fff;       // Disable ALL DMA channels
```

This is the critical phase:
- **Bit 15 (0x8000)** in these registers is the "set/clear" bit
- Writing **0x7fff** (bits 0-14 set, bit 15 clear) means "clear all these channels"
- The blitter, interrupts, and DMA are all disabled to prevent conflicts

### Color Reset & VBR Setup
```c
for (int a = 0; a < 32; a++)
    custom->color[a] = 0;      // Set all colors to black

VBR = GetVBR();                // Get Vector Base Register
SystemIrq = GetInterruptHandler(); // Save current interrupt handler
```

Resets the palette and saves the interrupt vector table location.

## System Restoration: FreeSystem()

Located at [main.c:128-158](main.c#L128-L158), this function restores normal OS operation.

### Cleanup Phase
```c
WaitVbl();
WaitBlit();
custom->intena = 0x7fff;       // Disable interrupts
custom->intreq = 0x7fff;       // Clear pending requests
custom->dmacon = 0x7fff;       // Disable DMA
```

Ensures everything is stopped cleanly before restoration.

### Restoration Phase
```c
SetInterruptHandler(SystemIrq);            // Restore interrupt handler
custom->cop1lc = (ULONG)GfxBase->copinit; // Restore system copper list
custom->cop2lc = (ULONG)GfxBase->LOFlist; // Restore second copper list
custom->copjmp1 = 0x7fff;                 // Restart copper

// Restore with SET bit (0x8000)
custom->intena = SystemInts | 0x8000;      // Re-enable saved interrupts
custom->dmacon = SystemDMA | 0x8000;       // Re-enable saved DMA
custom->adkcon = SystemADKCON | 0x8000;    // Re-enable audio/disk control
```

The **0x8000** bit means "set these bits" rather than "clear these bits".

### Final Cleanup
```c
WaitBlit();
DisownBlitter();               // Release blitter
Enable();                      // Re-enable task switching
LoadView(ActiView);            // Restore saved view
WaitTOF();
WaitTOF();
Permit();                      // Allow task switching
```

## Vector Base Register (VBR)

The VBR points to the interrupt vector table, which contains addresses of interrupt handlers.

### GetVBR() - [main.c:30-39](main.c#L30-L39)
```c
UWORD getvbr[] = {0x4e7a, 0x0801, 0x4e73};  // MOVEC.L VBR,D0 RTE

if (SysBase->AttnFlags & AFF_68010)
    vbr = (APTR)Supervisor((ULONG (*)())getvbr);
```

- Executes 68010+ CPU instruction `MOVEC VBR,D0` in supervisor mode
- On 68000, VBR is always at address 0
- On 68010+, it can be relocated

### SetInterruptHandler() - [main.c:41-44](main.c#L41-L44)
```c
*(volatile APTR *)(((UBYTE *)VBR) + 0x6c) = interrupt;
```

- Offset **0x6c** is the Level 3 Interrupt Autovector
- This is where vertical blank (VBLANK) interrupts vector to
- Writes the address of the new interrupt handler

### GetInterruptHandler() - [main.c:46-49](main.c#L46-L49)
```c
return *(volatile APTR *)(((UBYTE *)VBR) + 0x6c);
```

Reads the current interrupt handler address for later restoration.

## Custom Interrupt Handler

Located at [main.c:327-345](main.c#L327-L345), this handles vertical blank interrupts.

```c
static __attribute__((interrupt)) void interruptHandler()
{
    custom->intreq = (1 << INTB_VERTB);    // Clear VBLANK request
    custom->intreq = (1 << INTB_VERTB);    // Clear again (A4000 bug workaround)

    // Update scrolling parameters in copper list
    if (scroll)
    {
        // int sin = sinus15[frameCounter & 63];
        // *scroll = sin | (sin << 4);
    }

#ifdef MUSIC
    p61Music();                             // Call music player
#endif
    
    frameCounter++;                         // Increment frame counter
}
```

This runs 50 times per second (PAL) or 60 times per second (NTSC).

### Key Points:
- **Must clear the interrupt request** or it will keep firing
- **A4000 bug workaround**: clearing twice ensures reliability
- Perfect place for per-frame logic (music, counters, effects)
- Keep it **short** - long handlers cause timing issues

## Vertical Synchronization

### WaitVbl() - [main.c:53-71](main.c#L53-L71)
```c
void WaitVbl()
{
    // Wait until NOT at line 311
    while (1)
    {
        volatile ULONG vpos = *(volatile ULONG *)0xDFF004;
        vpos &= 0x1ff00;
        if (vpos != (311 << 8))
            break;
    }
    // Wait until AT line 311 (start of VBLANK)
    while (1)
    {
        volatile ULONG vpos = *(volatile ULONG *)0xDFF004;
        vpos &= 0x1ff00;
        if (vpos == (311 << 8))
            break;
    }
}
```

Reads the **VPOSR** register at **0xDFF004** to detect vertical position:
- **Bits 8-16** contain the vertical raster line number
- Line **311** is the start of vertical blank on PAL systems
- This ensures code runs once per frame

### WaitLine() - [main.c:73-81](main.c#L73-L81)
```c
void WaitLine(USHORT line)
{
    while (1)
    {
        volatile ULONG vpos = *(volatile ULONG *)0xDFF004;
        if (((vpos >> 8) & 511) == line)
            break;
    }
}
```

Waits for a specific raster line, used for precise timing within a frame.

### WaitBlt() - [main.c:83-90](main.c#L83-L90)
```c
inline void WaitBlt()
{
    UWORD tst = *(volatile UWORD *)&custom->dmaconr;  // A1000 compatibility
    (void)tst;
    while (*(volatile UWORD *)&custom->dmaconr & (1 << 14))
    {
        // Loop while blitter busy (bit 14)
    }
}
```

Polls **DMACONR** bit 14 (blitter busy flag) until the blitter finishes.

## Timing Reference

### PAL System (European):
- **312 lines** per frame
- **50 Hz** refresh rate
- VBLANK: lines **312-25** (wraps around)
- VSYNC: lines **2-5**

### NTSC System (US/Japan):
- **262 lines** per frame  
- **60 Hz** refresh rate
- Different VBLANK timing

### Main Loop Timing
The main loop at [main.c:496-539](main.c#L496-L539) uses:
```c
Wait10();  // Wait for line 0x10 (line 16)
```

This synchronizes blit operations to specific raster positions to avoid visual tearing.

## Interrupt Setup in main()

At [main.c:484-488](main.c#L484-L488):
```c
SetInterruptHandler((APTR)interruptHandler);
custom->intena = INTF_SETCLR | INTF_INTEN | INTF_VERTB;
#ifdef MUSIC
custom->intena = INTF_SETCLR | INTF_EXTER;  // Music player needs external interrupts
#endif
custom->intreq = (1 << INTB_VERTB);          // Clear any pending VBLANK
```

- **INTF_SETCLR** (0x8000): Set bits (rather than clear)
- **INTF_INTEN** (0x4000): Master interrupt enable
- **INTF_VERTB** (0x0020): Vertical blank interrupt
- **INTF_EXTER** (0x0008): External interrupt (used by some music players)

## Keyboard Input After TakeSystem()

When `TakeSystem()` disables all OS interrupts, it also disables the keyboard interrupt handler. This means the OS keyboard buffer is no longer updated and you need to handle keyboard input yourself.

### The Problem

After `TakeSystem()` executes:
- All interrupts are disabled (line [main.c:112](main.c#L112))
- Your custom interrupt handler only processes VERTB (vertical blank)
- The OS keyboard queue stops updating
- Normal OS input functions won't work

### Option 1: Direct Hardware Polling (Recommended)

Poll the CIA (Complex Interface Adapter) registers directly in your main loop. This is similar to how mouse input is already handled at [main.c:160-161](main.c#L160-L161):

```c
__attribute__((always_inline)) inline short MouseLeft() { 
    return !((*(volatile UBYTE *)0xbfe001) & 64); 
}
__attribute__((always_inline)) inline short MouseRight() { 
    return !((*(volatile UWORD *)0xdff016) & (1 << 10)); 
}
```

For keyboard, read the CIA-A registers:

```c
// CIA-A registers for keyboard
#define CIAA_SDR   ((volatile UBYTE *)0xBFEC01)  // Serial Data Register (keyboard scan codes)
#define CIAA_ICR   ((volatile UBYTE *)0xBFED01)  // Interrupt Control Register

inline UBYTE ReadKeyboard()
{
    UBYTE icr = *CIAA_ICR;  // Read interrupt control register
    if (icr & 0x08) {       // Check if SP (serial port) interrupt pending
        UBYTE keycode = *CIAA_SDR;  // Read the raw scancode
        return keycode;
    }
    return 0xFF;  // No key event
}
```

**Key Points:**
- Scan codes are **raw hardware codes**, not ASCII
- **Bit 7** indicates key up (1) or key down (0)
- You need a **scan code to key mapping table**
- Must handle key repeat yourself

### Option 2: Handle Keyboard Interrupts (More Complex)

Modify your interrupt handler to also process keyboard interrupts:

```c
// In main(), enable PORTS interrupt for keyboard:
SetInterruptHandler((APTR)interruptHandler);
custom->intena = INTF_SETCLR | INTF_INTEN | INTF_VERTB | INTF_PORTS;

// In interruptHandler():
static __attribute__((interrupt)) void interruptHandler()
{
    UWORD intreq = custom->intreqr;
    
    // Handle vertical blank
    if (intreq & INTF_VERTB) {
        custom->intreq = (1 << INTB_VERTB);
        custom->intreq = (1 << INTB_VERTB);  // A4000 bug workaround
        
        frameCounter++;
#ifdef MUSIC
        p61Music();
#endif
    }
    
    // Handle keyboard (ports interrupt)
    if (intreq & INTF_PORTS) {
        UBYTE keycode = *((volatile UBYTE *)0xBFEC01);  // Read CIA-A SDR
        // Process keycode...
        custom->intreq = INTF_PORTS;  // Clear the interrupt
    }
}
```

**Caution:** 
- Interrupt handlers must be **short and fast**
- Don't do complex processing inside the handler
- Store the keycode in a buffer for main loop processing

### Option 3: Use OS Before TakeSystem()

If you only need keyboard input for initial configuration (like menu selection), read it **before** calling `TakeSystem()`:

```c
// Before TakeSystem() - OS is still active
Write(Output(), (APTR)"Press key to start...\n", 22);
char buffer[2];
Read(Input(), buffer, 1);  // Wait for keypress

// Now take system
TakeSystem();
// ... rest of demo ...
```

### Scan Code Reference

Common Amiga raw keyboard scan codes (key down, bit 7 = 0):

| Key | Scan Code | Key | Scan Code |
|-----|-----------|-----|-----------|
| ESC | 0x45 | Space | 0x40 |
| F1-F10 | 0x50-0x59 | Return | 0x44 |
| Arrow Up | 0x4C | Arrow Down | 0x4D |
| Arrow Left | 0x4F | Arrow Right | 0x4E |
| A-Z | 0x20-0x39 | 0-9 | 0x01-0x0A |

Key **up** events have bit 7 set (add 0x80 to the key down code).

### Recommendation

For most demo scenarios, **use Option 1 (polling)**. It's:
- Simpler to implement and debug
- More predictable timing
- Sufficient for checking specific keys (ESC to quit, arrows for control, etc.)
- Consistent with how mouse input is already handled

Only use interrupt-based input if you need to capture every single keypress without polling overhead.

## Best Practices

1. **Always restore the system** - Use FreeSystem() even if your program crashes
2. **Keep interrupt handlers short** - Long handlers cause timing glitches
3. **Clear interrupt requests** - Always clear the bit that caused the interrupt
4. **Wait for blitter** - Call WaitBlt() before starting new blitter operations
5. **Synchronize with video** - Use WaitVbl() to avoid tearing
6. **A4000 compatibility** - Clear interrupt requests twice
7. **Keyboard input** - Poll CIA registers or handle PORTS interrupt yourself

## Common Pitfalls

- **Forgetting to disable interrupts** before modifying interrupt vectors → crashes
- **Not clearing interrupt requests** → interrupt fires continuously
- **Not waiting for blitter** → corrupted graphics
- **Long interrupt handlers** → audio glitches, timing issues
- **Not restoring system state** → OS unusable after program exits
