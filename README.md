# UltraBoost

**The Ultimate Co-Processor for Amstrad CPC (464/6128/Plus)**

UltraBoost is a fully backward-compatible external expansion card based on the Raspberry Pi RP2350B. It transparently replaces/emulates RAM, ROM, and the WD1772 FDC while adding high-performance graphics acceleration, mass storage, and future enhancements (VGA, sound, USB HID).

100% software compatible — no modifications required.

## Hardware

- **Core**: Waveshare Core2350B (RP2350B dual Arm Cortex-M33 @150–300 MHz, 520 KB SRAM, 8 MB PSRAM)
- **Bus Interface**: 74HC245 bidirectional buffer (5 V tolerant)
- **Connection**: MX4 expansion port ribbon + CSYNC from monitor DIN-4
- **Peripherals**:
  - microSD (SPI) for ROMs/.DSK/.HFE
  - Headers for I2S audio, VGA/RGB, USB, SWD, I2C
- **Power**: 5 V from CPC expansion port
- **Size**: Compact 2-layer PCB (~50×60 mm)

## Core Features (Transparent Emulation)

- **RAM Emulation**: 4 MB total (128 KB fast SRAM visible, rest PSRAM-backed), full 6128 banking
- **ROM Emulation**: Multi-slot lower/upper, hot-load from SD, UltraBoost system ROM on boot
- **FDC Emulation**: Full WD1772, .DSK/.HFE/extended formats from microSD
- **I/O Snooping**: Gate Array (&7Fxx), CRTC (&BC/&BD) for mode/base awareness
- **Mass Storage**: Direct file access, AMSDOS/PARADOS switching

## Graphics Acceleration (Opt-in via RSX / Jumptable)

Hybrid double-buffering (32 KB private back buffer + VSYNC swap):

- Accelerated LDIR/LDDR, RLE/compressed transfers
- Primitives: plot, line (thick/dashed), circle/ellipse (filled), rectangle (solid/pattern/gradient), polygon fill
- Blits: arbitrary source/dest, logical ops, transparency, flip/mirror, scaling (1–8× bilinear), 360° rotation/shear
- Dithering: 256–4096 apparent colours
- Command queue + RSX calls (|BLIT, |LINE, |FILL, etc.)
- ASM direct jumptable for machine code

## Performance / Speed Increases

- **Bus Mastering**: Cycle-accurate, zero added wait states (PIO + DMA + 300 MHz overclock)
- **RAM/ROM Access**: Zero-wait in SRAM (first 128 KB), no contention penalties
- **Graphics**:
  - Blits/LDIR: 10–100× faster than Z80 software
  - Primitives: 50–500× speedup vs hand-coded Z80 routines
  - Double-buffer: Flicker-free updates, dirty-rectangle optimised
  - Complex effects (particles, plasma, scaling) practical in real-time
- **Storage**: Instant disk access from SD vs mechanical floppy (1000×+)
- **Overall**: Enables modern demos, SymbOS acceleration, large assets without slowdown

## Compatibility

- Amstrad CPC 464/6128/Plus
- RAMDIS disables onboard 6128 RAM (full control) 464 only blocks reads, but works not issue
- All original software runs unchanged
- Acceleration activated via magic signature/RSX

## Status & Next Steps

- PCB designed, veroboard prototype in progress
- Core PIO/ARM bus master code ready
- Basic RAM/ROM/FDC functional
- Graphics queue + primitives in development

## Future / Planned

- VGA scan-doubler / high-res modes
- Enhanced sound (I2S + OPL4 emulation)
- USB HID host (modern input)
- Symbiface II emulation (IDE/RTC/mouse)

![UltraBoost PCB Render](ultraboost_pcb_render.png)

