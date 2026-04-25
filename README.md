# UltraBoost - Complete Feature Summary!

![UltraBoost PCB Render](ultraboost_pcb_render.png)

*pcb image is not final!

# CPC V9990 Accelerator Card — Design Reference v7

---

## Quick reference

| Item | Detail |
|------|--------|
| MCU module | Waveshare Core2350B — RP2350B + 8 MB PSRAM + 16 MB flash |
| System clock | 294 MHz (PLL_SYS exact — 12 × 49 ÷ 2) |
| Bus transceiver | 74LVC245 DIP-20 — the only external logic IC |
| Colour output | RGB555 — 15-bit, 32,768 colours |
| CPC compatibility | 464 / 664 / 6128 — card provides 128 KB RAM and 6128 ROM set |
| V9990 ports | &FF60–&FF6F — GFX9000 compatible, SymbOS driver unmodified |
| Coprocessor ports | &FF40–&FF4F — math, 3D rasteriser |
| Accelerator ROM | CPC upper ROM slot 5 — RSX auto-installs at boot |
| CPC connector | 50-pin right angle 2×25 header, 2.54 mm pitch, MX4 compatible |
| Reserved GPIO | GPIO0 (boot select) · GPIO47 (PSRAM CS — SDK managed) |
| Active GPIO | GPIO1–46 = 46 pins, no gaps |

---

# PART 1 — HARDWARE DESIGN

---

## 1. What the Card Does

The card plugs into the CPC 50-pin expansion connector via an MX4-compatible backplane. From the CPC's perspective it presents four things simultaneously:

**1. 128 KB RAM** — all 8 memory banking configurations of the CPC 6128 are emulated in the RP2350B's internal SRAM. The CPC's internal RAM chips are disabled via RAMDIS. A CPC 464 gains 6128-compatible extended memory.

**2. A full CPC 6128 ROM set** — the 6128 BASIC ROM (slot 0), AMSDOS ROM (slot 7), and our accelerator ROM (slot 5) are served from RP2350B flash. The CPC's own ROM chips are permanently disabled via ROMDIS (hardwired HIGH). A CPC 464 boots into 6128 BASIC with full AMSDOS disc support.

**3. A V9990 VDP** at ports &FF60–&FF6F — electrically equivalent to a GFX9000 card. SymbOS's existing GFX9000 CPC driver works without modification. Full V9990 VRAM emulation in 8 MB PSRAM.

**4. A math and 3D coprocessor** at ports &FF40–&FF4F — hardware multiply, trig lookup, matrix transforms, perspective divide, and textured 3D triangle rasterisation running asynchronously on Core 1.

On power-on the card immediately boots into CPC native display mode — reconstructing the CPC screen from emulated RAM and driving it to VGA at 50 Hz with a cycle-accurate CRTC state machine. The user sees the BASIC start screen on their VGA monitor from the very first frame.


```svg
<svg width="100%" viewBox="0 0 1800 520" xmlns="http://www.w3.org/2000/svg" style="font-family:sans-serif;background:#1a1a2e">
  <defs>
    <marker id="arr" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="#aaa" stroke-width="1.5" stroke-linecap="round"/>
    </marker>
  </defs>

  <!-- CPC Connector -->
  <rect x="20" y="40" width="180" height="440" rx="8" fill="#2d2d44" stroke="#6c6caa" stroke-width="1.5"/>
  <text x="110" y="68" text-anchor="middle" fill="#aac" font-size="16" font-weight="bold">CPC 50-pin</text>
  <text x="110" y="88" text-anchor="middle" fill="#888" font-size="13">MX4 backplane</text>
  <rect x="35" y="106" width="150" height="34" rx="4" fill="#3a3a5c" stroke="#6699aa" stroke-width="1"/>
  <text x="110" y="128" text-anchor="middle" fill="#9cf" font-size="13">A0–A15 (address)</text>
  <rect x="35" y="148" width="150" height="34" rx="4" fill="#3a3a5c" stroke="#aa9933" stroke-width="1"/>
  <text x="110" y="170" text-anchor="middle" fill="#fc9" font-size="13">D0–D7 (data)</text>
  <rect x="35" y="190" width="150" height="34" rx="4" fill="#3a3a5c" stroke="#33aa66" stroke-width="1"/>
  <text x="110" y="212" text-anchor="middle" fill="#9fb" font-size="13">/MREQ /WR /RESET</text>
  <rect x="35" y="232" width="150" height="34" rx="4" fill="#3a3a5c" stroke="#aa3333" stroke-width="1"/>
  <text x="110" y="254" text-anchor="middle" fill="#f99" font-size="13">RAMDIS (hardwired HI)</text>
  <rect x="35" y="274" width="150" height="34" rx="4" fill="#3a3a5c" stroke="#aa3333" stroke-width="1"/>
  <text x="110" y="296" text-anchor="middle" fill="#f99" font-size="13">ROMDIS (hardwired HI)</text>
  <rect x="35" y="316" width="150" height="34" rx="4" fill="#3a3a5c" stroke="#888" stroke-width="1"/>
  <text x="110" y="338" text-anchor="middle" fill="#aaa" font-size="13">+5 V · GND</text>
  <rect x="35" y="420" width="150" height="40" rx="4" fill="#3a1a1a" stroke="#aa4444" stroke-width="1"/>
  <text x="110" y="436" text-anchor="middle" fill="#f66" font-size="13">NO /WAIT</text>
  <text x="110" y="452" text-anchor="middle" fill="#f66" font-size="11">not connected</text>

  <!-- 74LVC245 -->
  <rect x="240" y="148" width="130" height="100" rx="8" fill="#3a3300" stroke="#aa9933" stroke-width="1.5"/>
  <text x="305" y="185" text-anchor="middle" fill="#fc9" font-size="15" font-weight="bold">74LVC245</text>
  <text x="305" y="205" text-anchor="middle" fill="#aa8" font-size="12">5 V supply</text>
  <text x="305" y="222" text-anchor="middle" fill="#aa8" font-size="12">DIR · /OE</text>
  <text x="305" y="239" text-anchor="middle" fill="#aa8" font-size="12">DIP-20</text>

  <!-- Core2350B -->
  <rect x="420" y="20" width="860" height="480" rx="10" fill="#1e1e3a" stroke="#6666cc" stroke-width="2"/>
  <text x="850" y="52" text-anchor="middle" fill="#99f" font-size="18" font-weight="bold">Waveshare Core2350B</text>
  <text x="850" y="74" text-anchor="middle" fill="#667" font-size="13">RP2350B · 294 MHz · dual Cortex-M33 · 3.3 V LDO onboard</text>

  <!-- PIO0 -->
  <rect x="440" y="92" width="400" height="70" rx="6" fill="#1a2a3a" stroke="#3399cc" stroke-width="1.2"/>
  <text x="640" y="118" text-anchor="middle" fill="#6cf" font-size="15" font-weight="bold">PIO Block 0 — bus interface</text>
  <text x="640" y="138" text-anchor="middle" fill="#558" font-size="12">SM0: write snooper  ·  SM1: memory read handler</text>
  <text x="640" y="154" text-anchor="middle" fill="#558" font-size="12">in pins,32 captures entire bus state in one instruction</text>

  <!-- PIO1 -->
  <rect x="440" y="172" width="400" height="56" rx="6" fill="#1a2a3a" stroke="#33cc99" stroke-width="1.2"/>
  <text x="640" y="196" text-anchor="middle" fill="#6fc" font-size="15" font-weight="bold">PIO Block 1 — VGA output</text>
  <text x="640" y="214" text-anchor="middle" fill="#558" font-size="12">SM0: HSYNC/VSYNC  ·  SM1: RGB555 pixel serialiser</text>

  <!-- Core 0 -->
  <rect x="440" y="240" width="190" height="80" rx="6" fill="#2a1a1a" stroke="#cc6633" stroke-width="1.2"/>
  <text x="535" y="270" text-anchor="middle" fill="#fa8" font-size="15" font-weight="bold">Core 0</text>
  <text x="535" y="288" text-anchor="middle" fill="#876" font-size="12">bus handler</text>
  <text x="535" y="306" text-anchor="middle" fill="#876" font-size="12">ROM · RAM · port decode</text>

  <!-- Core 1 -->
  <rect x="650" y="240" width="190" height="80" rx="6" fill="#2a1a1a" stroke="#cc6633" stroke-width="1.2"/>
  <text x="745" y="270" text-anchor="middle" fill="#fa8" font-size="15" font-weight="bold">Core 1</text>
  <text x="745" y="288" text-anchor="middle" fill="#876" font-size="12">V9990 + CRTC engine</text>
  <text x="745" y="306" text-anchor="middle" fill="#876" font-size="12">3D rasteriser</text>

  <!-- SRAM -->
  <rect x="440" y="334" width="400" height="54" rx="6" fill="#1a1a2a" stroke="#666688" stroke-width="1.2"/>
  <text x="640" y="358" text-anchor="middle" fill="#99a" font-size="14" font-weight="bold">520 KB SRAM (internal)</text>
  <text x="640" y="376" text-anchor="middle" fill="#445" font-size="12">128 KB CPC RAM · V9990 regs · line buffers · command ring</text>

  <!-- Flash -->
  <rect x="440" y="398" width="400" height="46" rx="6" fill="#1a1a2a" stroke="#556655" stroke-width="1.2"/>
  <text x="640" y="420" text-anchor="middle" fill="#8a8" font-size="14" font-weight="bold">16 MB Flash (QSPI0 — dedicated)</text>
  <text x="640" y="438" text-anchor="middle" fill="#445" font-size="12">firmware · 6128 BASIC (16 KB) · AMSDOS (16 KB) · accel ROM slot 5 (16 KB)</text>

  <!-- PSRAM -->
  <rect x="860" y="92" width="400" height="352" rx="6" fill="#1a2a1a" stroke="#339966" stroke-width="1.2"/>
  <text x="1060" y="120" text-anchor="middle" fill="#6c9" font-size="15" font-weight="bold">8 MB PSRAM (QSPI1)</text>
  <text x="1060" y="140" text-anchor="middle" fill="#456" font-size="12">GPIO47 CS — SDK managed</text>
  <rect x="876" y="152" width="368" height="40" rx="4" fill="#1a3a2a" stroke="#336644"/>
  <text x="1060" y="177" text-anchor="middle" fill="#8fb" font-size="13">V9990 VRAM — 512 KB (0x000000)</text>
  <rect x="876" y="198" width="368" height="40" rx="4" fill="#1a3a2a" stroke="#336644"/>
  <text x="1060" y="223" text-anchor="middle" fill="#8fb" font-size="13">VGA framebuffer — 2 MB (0x080000)</text>
  <rect x="876" y="244" width="368" height="40" rx="4" fill="#1a3a2a" stroke="#336644"/>
  <text x="1060" y="269" text-anchor="middle" fill="#8fb" font-size="13">Z-buffer — 512 KB (0x280000)</text>
  <rect x="876" y="290" width="368" height="40" rx="4" fill="#1a3a2a" stroke="#336644"/>
  <text x="1060" y="315" text-anchor="middle" fill="#8fb" font-size="13">Texture RAM — 2 MB (0x300000)</text>
  <rect x="876" y="336" width="368" height="40" rx="4" fill="#1a3a2a" stroke="#336644"/>
  <text x="1060" y="361" text-anchor="middle" fill="#8fb" font-size="13">Backing store — 3 MB (0x500000)</text>
  <text x="1060" y="440" text-anchor="middle" fill="#456" font-size="11">GPIO0 reserved (boot select)</text>

  <!-- VGA DAC -->
  <rect x="1320" y="20" width="460" height="290" rx="8" fill="#1a2a1a" stroke="#44aa44" stroke-width="1.5"/>
  <text x="1550" y="52" text-anchor="middle" fill="#6d6" font-size="17" font-weight="bold">VGA RGB555 DAC</text>
  <text x="1550" y="74" text-anchor="middle" fill="#484" font-size="13">15 GPIO → 32,768 colours</text>
  <text x="1340" y="108" fill="#595" font-size="13">B0–B4 (GPIO32–36)  10k/4.7k/2.2k/1k/560Ω → pin 3</text>
  <text x="1340" y="132" fill="#595" font-size="13">G0–G4 (GPIO37–41)  identical ladder → pin 2</text>
  <text x="1340" y="156" fill="#595" font-size="13">R0–R4 (GPIO42–46)  identical ladder → pin 1</text>
  <text x="1340" y="188" fill="#595" font-size="13">HSYNC (GPIO30) → DE-15 pin 13</text>
  <text x="1340" y="208" fill="#595" font-size="13">VSYNC (GPIO31) → DE-15 pin 14</text>
  <text x="1340" y="228" fill="#595" font-size="13">GND             → DE-15 pins 5,6,7,8,10</text>
  <rect x="1390" y="248" width="320" height="44" rx="6" fill="#1a3a1a" stroke="#339933"/>
  <text x="1550" y="275" text-anchor="middle" fill="#8f8" font-size="15" font-weight="bold">DE-15 VGA connector</text>

  <!-- VGA modes summary -->
  <rect x="1320" y="330" width="460" height="170" rx="8" fill="#1a2a2a" stroke="#33aaaa" stroke-width="1.5"/>
  <text x="1550" y="358" text-anchor="middle" fill="#6cc" font-size="15" font-weight="bold">VGA output modes</text>
  <text x="1340" y="384" fill="#456" font-size="13">Mode 0: CPC native 50 Hz (CRTC reconstruction)</text>
  <text x="1340" y="404" fill="#456" font-size="13">Modes 1–33: direct framebuffer (all fit in 2 MB)</text>
  <text x="1340" y="424" fill="#456" font-size="13">From 160×120 up to 1920×270 → 1920×1080</text>
  <text x="1340" y="444" fill="#456" font-size="13">PLL_USB reprogrammed per mode — exact clocks</text>
  <text x="1340" y="464" fill="#456" font-size="13">PLL_SYS stays at 294 MHz — bus unaffected</text>
  <text x="1340" y="487" fill="#456" font-size="13">★ double-scan modes recommended for XGA/720p/1080p</text>

  <!-- Arrows -->
  <!-- A bus: CPC → Core2350B -->
  <line x1="200" y1="123" x2="420" y2="123" stroke="#6699aa" stroke-width="2" marker-end="url(#arr)"/>
  <text x="310" y="113" text-anchor="middle" fill="#9cf" font-size="11">GPIO 1–16</text>

  <!-- D bus: CPC ↔ 245 ↔ Core2350B -->
  <line x1="200" y1="165" x2="240" y2="165" stroke="#aa9933" stroke-width="2" marker-end="url(#arr)" marker-start="url(#arr)"/>
  <line x1="370" y1="198" x2="420" y2="198" stroke="#aa9933" stroke-width="2" marker-end="url(#arr)" marker-start="url(#arr)"/>
  <text x="395" y="190" text-anchor="middle" fill="#fc9" font-size="11">GPIO 17–24</text>

  <!-- DIR/OE -->
  <line x1="420" y1="230" x2="370" y2="230" stroke="#665" stroke-width="1.5" marker-end="url(#arr)"/>
  <text x="395" y="222" text-anchor="middle" fill="#996" font-size="10">GPIO 25–26</text>

  <!-- Control: CPC → Core2350B -->
  <line x1="200" y1="207" x2="420" y2="155" stroke="#33aa66" stroke-width="2" marker-end="url(#arr)"/>
  <text x="295" y="178" text-anchor="middle" fill="#9fb" font-size="11">GPIO 27–29</text>

  <!-- VGA colour -->
  <line x1="1280" y1="130" x2="1320" y2="130" stroke="#44aa44" stroke-width="2" marker-end="url(#arr)"/>
  <text x="1300" y="120" text-anchor="middle" fill="#8d8" font-size="11">GPIO 32–46</text>

  <!-- VGA sync -->
  <line x1="1280" y1="185" x2="1320" y2="185" stroke="#44aa44" stroke-width="2" marker-end="url(#arr)"/>
  <text x="1300" y="175" text-anchor="middle" fill="#8d8" font-size="11">GPIO 30–31</text>

  <!-- PSRAM -->
  <line x1="840" y1="300" x2="860" y2="300" stroke="#339966" stroke-width="2" marker-end="url(#arr)" marker-start="url(#arr)"/>
</svg>
```


---

## 2. Why No /WAIT

At 294 MHz the RP2350B has over 400 ns of margin to respond to any Z80 memory read:

```
Z80 at 4 MHz — data must be valid within ~500 ns of /MREQ falling

RP2350B response chain at 294 MHz:
  Address bus propagation to GPIO (direct, no resistors)   ~4 ns
  PIO detects /MREQ + /WR high, pushes address to FIFO    ~14 ns
  Core 0 IRQ entry (12 cycles @ 294 MHz)                  ~41 ns
  SRAM lookup (single cycle, cache-hot)                     ~3 ns
  GPIO drive via sio_hw->gpio_out                           ~3 ns
  74LVC245 propagation + PCB                               ~8 ns
  ──────────────────────────────────────────────────────────────
  Total                                                    ~73 ns
  Margin before Z80 samples data:  500 - 73 = 427 ns  ✓
```

For Z80 I/O cycles the margin is ~930 ns. All coprocessor math completes faster than the Z80 can write the operand bytes. **No /WAIT signal is used or connected anywhere on this card.**

---

## 3. Signal Reduction — Three Control Signals

All Z80 bus cycle types are completely identified using just /MREQ and /WR:

| /MREQ | /WR | Cycle type | Our action |
|-------|-----|-----------|-----------|
| 0 | 1 | Memory read (including opcode fetch) | Serve RAM or ROM byte |
| 0 | 0 | Memory write | Store to emulated RAM |
| 1 | 0 | I/O write — OUT (C),r | Decode port address |
| 1 | 1 | I/O read, IRQ ack, or idle | Pre-loaded status |

**/IORQ dropped** — I/O write = /MREQ high + /WR low. Unambiguous.
**/RD dropped** — memory read = /MREQ low + /WR high. Unambiguous.
**/M1 dropped** — interrupt acknowledge has /WR high so cannot trigger write handler.
**DRAM refresh** — /MREQ low + /WR high, treated as memory read. Z80 discards the data. Harmless on SRAM.

The three freed GPIO pins are used for the RGB555 fifth colour bit per channel.

**INTACK approximation:** Without /IORQ and /M1 we cannot detect the Z80 interrupt acknowledge cycle directly. We model it as occurring 1 µs after /INT is asserted — accurate to ±1 scanline for the vast majority of demos and games. See section 13 for details.

---

## 4. GPIO Map

All bus-facing signals occupy GPIO1–29, routing toward the CPC connector. All VGA signals occupy GPIO30–46, routing toward the DE-15. Clean physical separation for PCB layout.

### Group A — CPC Address Bus (GPIO1–16)

| GPIO | Signal | Dir | Notes |
|------|--------|-----|-------|
| 1–16 | A0–A15 | IN | Direct to RP2350B. PIO IN_BASE=GPIO1. No series resistors — RP2350B inputs are 5 V tolerant for reading. Z80 always drives these — no pull-downs needed. |

### Group B — CPC Data Bus via 74LVC245 (GPIO17–24)

| GPIO | Signal | Dir | Notes |
|------|--------|-----|-------|
| 17–24 | D0–D7 | BIDIR | Through 74LVC245 (5 V supply from CPC). CPC side is native 5 V. RP2350B side is 3.3 V — handled by LVC245 output stage. Direction: GPIO25. Enable: GPIO26. PIO OUT_BASE=GPIO17. |

### Group C — 74LVC245 Bus Transceiver Control (GPIO25–26)

| GPIO | Signal | Dir | Notes |
|------|--------|-----|-------|
| 25 | 245 DIR | OUT | HIGH = RP2350B drives D0–D7 (ROM/RAM reads). LOW = CPC drives D0–D7 (writes). Default LOW. |
| 26 | 245 /OE | OUT | LOW = bus active. HIGH = high-Z (disconnected). Default HIGH. |

### Group D — Z80 Control Signals (GPIO27–29)

| GPIO | Signal | Dir | Notes |
|------|--------|-----|-------|
| 27 | /MREQ  | IN | 10 kΩ pull-up to 3.3 V. PIO wait target for memory read SM. |
| 28 | /WR    | IN | 10 kΩ pull-up. PIO JMP_PIN — write snooper fires on falling edge. |
| 29 | /RESET | IN | 10 kΩ pull-up. Triggers full card reinitialisation on CPC reset. |

### Group E — VGA Sync (GPIO30–31)

| GPIO | Signal | Dir | Notes |
|------|--------|-----|-------|
| 30 | HSYNC | OUT | PIO1 SM0. Active low. DE-15 pin 13. 3.3 V direct — no resistor. |
| 31 | VSYNC | OUT | PIO1 SM0. Active low. DE-15 pin 14. |

### Group F — RGB555 Colour DAC (GPIO32–46)

PIO OUT_BASE=GPIO32. `out pins,15` drives one complete pixel. Two pixels packed per 32-bit DMA word (15 + 15 + 2 padding bits).

| GPIO | Signal | Resistor | Notes |
|------|--------|---------|-------|
| 32 | B0 LSB | 10 kΩ | → VGA Blue (DE-15 pin 3) |
| 33 | B1 | 4.7 kΩ | |
| 34 | B2 | 2.2 kΩ | |
| 35 | B3 | 1.0 kΩ | |
| 36 | B4 MSB | 560 Ω | |
| 37 | G0 LSB | 10 kΩ | → VGA Green (DE-15 pin 2) |
| 38 | G1 | 4.7 kΩ | |
| 39 | G2 | 2.2 kΩ | |
| 40 | G3 | 1.0 kΩ | |
| 41 | G4 MSB | 560 Ω | |
| 42 | R0 LSB | 10 kΩ | → VGA Red (DE-15 pin 1) |
| 43 | R1 | 4.7 kΩ | |
| 44 | R2 | 2.2 kΩ | |
| 45 | R3 | 1.0 kΩ | |
| 46 | R4 MSB | 560 Ω | |

**Total: 16 + 8 + 2 + 3 + 2 + 15 = 46 GPIO. GPIO1–GPIO46, no gaps.**

---

## 5. RGB555 DAC

Binary-weighted resistor ladder. 3.3 V GPIO into 75 Ω VGA termination (inside the monitor).

```
GPIO_B4 MSB ──[ 560 Ω]──┐
GPIO_B3     ──[1.0 kΩ]──┤
GPIO_B2     ──[2.2 kΩ]──┼──── VGA Blue (pin 3) ──── 75 Ω → GND (in monitor)
GPIO_B1     ──[4.7 kΩ]──┤
GPIO_B0 LSB ──[10  kΩ]──┘

Green and Red: identical ladder → VGA pins 2 and 1
HSYNC direct 3.3 V ──── DE-15 pin 13
VSYNC direct 3.3 V ──── DE-15 pin 14
GND         ──────────── DE-15 pins 5, 6, 7, 8, 10
```

All 5 bits high: ≈ 0.694 V into 75 Ω (VGA spec max 0.7 V ✓). LSB step ≈ 0.022 V.
Use 1% tolerance axial resistors. Place all 15 close to the DE-15 connector.

---

## 6. RAMDIS and ROMDIS

Both signals are **hardwired permanently HIGH** via resistors to +5 V (from CPC bus). This is the same approach as many CPC expansion devices.

```
+5 V ──[4.7 kΩ]──── CPC expansion pin 45  (RAMDIS)
+5 V ──[4.7 kΩ]──── CPC expansion pin 44  (ROMDIS)
```

**RAMDIS HIGH** disables the CPC's internal RAM chips. We serve all 128 KB from our SRAM.
**ROMDIS HIGH** disables all CPC ROM chips — BASIC on motherboard, AMSDOS on disc interface. We serve all ROM slots from our flash. No bus conflict is possible.

---

## 7. Hardware BOM

| Qty | Part | Package | Notes |
|-----|------|---------|-------|
| 1 | Waveshare Core2350B (8 MB PSRAM) | Stamp module, 2.54 mm headers | Main MCU. Onboard LDO, 16 MB flash, 8 MB PSRAM. |
| 1 | 74LVC245 | DIP-20 | Data bus transceiver. Powered from CPC +5 V. Socket recommended. |
| 3 | 560 Ω 1% | Axial 1/4 W | VGA R4/G4/B4 MSB |
| 3 | 1.0 kΩ 1% | Axial 1/4 W | VGA R3/G3/B3 |
| 3 | 2.2 kΩ 1% | Axial 1/4 W | VGA R2/G2/B2 |
| 3 | 4.7 kΩ 1% | Axial 1/4 W | VGA R1/G1/B1 |
| 3 | 10 kΩ 1% | Axial 1/4 W | VGA R0/G0/B0 LSB |
| 2 | 4.7 kΩ | Axial 1/4 W | RAMDIS + ROMDIS pull-up to +5 V |
| 3 | 10 kΩ | Axial 1/4 W | Pull-ups on /MREQ, /WR, /RESET |
| 3 | 100 nF | Disc ceramic 2.54 mm | Decoupling: VBUS, 3.3 V rail, 74LVC245 VCC |
| 2 | 10 µF 16 V | Radial electrolytic | Bulk decoupling: VBUS rail, 3.3 V rail |
| 1 | DE-15 female | THT PCB mount | VGA output |
| 1 | 50-pin right angle header | 2×25, 2.54 mm | CPC bus — plugs into MX4 backplane |
| 1 | 2-pin header + jumper | 2.54 mm | Data bus disconnect for development |
| 1 | Tactile button | THT 6×6 mm | BOOTSEL — GPIO0 to GND for firmware update |

**ROM images required:** CPC 6128 BASIC (16 KB) and AMSDOS (16 KB) — stored in RP2350B flash. Amstrad have granted free non-commercial use permission for CPC ROM images.

---

## 8. PCB Design Notes

- **CPC connector:** Right angle 2×25 header on the top edge of the board, pins pointing horizontally into the MX4 backplane socket. All components on the same face. Verify MX4 pinout before ordering.
- **Card dimensions:** Check MX4 backplane slot pitch before finalising the board outline.
- **74LVC245:** Place as close to the CPC connector as possible. Keep data bus stub length under 20 mm. VCC decoupling cap directly at pin 20.
- **Address bus:** A0–A15 connect direct — no series resistors. RP2350B inputs are 5 V tolerant for reading and the Z80 always drives these lines.
- **Control signals:** 33 Ω series resistors on /MREQ, /WR, /RESET are optional. Omit for best timing.
- **RAMDIS / ROMDIS:** 4.7 kΩ pull-up resistors to the CPC +5 V rail. Place near the connector.
- **VGA resistors:** All 15 close to the DE-15. Separate ground region under DAC area.
- **Decoupling:** 100 nF ceramic on VBUS, 3.3 V pin, and 74LVC245 VCC. 10 µF electrolytic on VBUS and 3.3 V rails.
- **Development jumper:** 2-pin header to break the data bus — allows testing Core2350B independently without CPC.
- **BOOTSEL:** Tactile button from GPIO0 to GND for firmware update mode.
- **GPIO47:** Do not connect to anything — PSRAM CS is wired internally on the Core2350B module.


---

# PART 2 — CPC BUS INTERFACE

---

## 9. CPC I/O Port Architecture

The CPC places the full 16-bit address on the bus during I/O cycles. `OUT (C),r` puts the **B register** on A15–A8 (device select) and the **C register** on A7–A0 (register select).

**Why &FFxx is clean:** `&FF` = `1111 1111`. Every CPC internal device responds when one specific address bit is low. Since all bits in &FF are high, no internal hardware responds:

| Address bit | CPC device | Trigger | Safe? |
|-------------|-----------|---------|-------|
| A15 | Gate Array | A15=0 | ✓ |
| A14 | CRTC 6845  | A14=0 | ✓ |
| A13 | ROM select | A13=0 | ✓ |
| A12 | Printer    | A12=0 | ✓ |
| A11 | PPI 8255   | A11=0 | ✓ |
| A10 | Expansion  | A10=0 | ✓ |

Verified against the full CPCWiki I/O port table. No conflicts with SYMBiFACE (&FD00–&FD4F), M4 Board (&FC00, &FE00), or Albireo (&FE80–&FEBF).

**&FCxx is not used** — conflicts with M4 Board ACK/KICK mechanism.

### Z80 coding rules

```asm
; ALWAYS use OUT (C),r  — B = &FF (device), C = register
    LD B, &FF
    LD C, &60       ; V9990 VRAM data port
    LD A, data
    OUT (C), A      ; B:C = &FF60

; NEVER use OTIR or OTDR — they decrement B, corrupting device select
; NEVER use OUT (n),A for our ports

; Streaming bytes — use D as counter, never B:
SEND_LOOP:
    LD A, (HL) : INC HL
    OUT (C), A      ; B stays &FF throughout
    DEC D : JR NZ, SEND_LOOP
```

---

## 10. V9990 Port Map — &FF60 to &FF6F

GFX9000 compatible. SymbOS GFX9000 CPC driver works unmodified. V9990 palette uses RGB333 format (3 bits per channel = 512 palette colours) — our firmware converts to RGB555 for DAC output. B1 bitmap mode uses RGB555 pixels directly in VRAM, bypassing the palette.

| Port | B:C | Dir | Function |
|------|-----|-----|---------|
| &FF60 | &FF:&60 | R/W | VRAM data — auto-increments address pointer on each access |
| &FF61 | &FF:&61 | W | Palette data — RGB333 format, auto-incrementing |
| &FF63 | &FF:&63 | R/W | Register data — reads/writes register selected by &FF64 |
| &FF64 | &FF:&64 | W=select / R=status | Write: select register 0–63. Read: CE TR EO BD VR HR flags |
| &FF65 | &FF:&65 | W | Command data — blitter command parameters |
| &FF67 | &FF:&67 | W | System control — reset, XTAL, VRAM size |
| &FF6F | &FF:&6F | W | Output/interrupt control — display enable, IRQ enables |

### V9990 screen modes

| Mode | Resolution | Depth | VGA output | Notes |
|------|-----------|-------|-----------|-------|
| P1 (2-plane) | 256×212 | 4bpp, 61 colours | 640×480@60 Hz | SymbOS desktop |
| B1 (bitmap) | 256×212 | 16bpp RGB555 | 640×480@60 Hz | Full colour |
| B3 (bitmap) | 512×212 | 8bpp, 256 col | 640×480@60 Hz | Wide bitmap |
| B5 (bitmap) | 768×212 | 4bpp | 800×600@60 Hz | |
| B7 (bitmap) | 1024×212 | 4bpp | 1024×768@60 Hz | |

---

## 11. Coprocessor Port Map — &FF40 to &FF4F

Verified clean gap: above CPC Booster (&FF00–&FF28), below V9990 (&FF60). Nothing assigned in this range in the full CPCWiki port list.

| Port | B:C | Dir | Function |
|------|-----|-----|---------|
| &FF40 | &FF:&40 | W | CMD — write command opcode |
| &FF41 | &FF:&41 | W | DATA_IN — stream operand bytes after CMD |
| &FF42 | &FF:&42 | R | DATA_OUT — read result bytes after BUSY clears |
| &FF43 | &FF:&43 | R | STATUS — bit0=BUSY bit1=READY bit2=OVF bit3=DIV0 bit7=QFULL |
| &FF44–&FF46 | W | ZBUF_ADDR — 24-bit PSRAM address of Z-buffer (3 bytes LSB first) |
| &FF47–&FF49 | W | TEX_ADDR — 24-bit PSRAM address of current texture (3 bytes LSB first) |
| &FF4A–&FF4F | — | Reserved |

---

## 12. Coprocessor Command Set

Poll &FF43 STATUS bit 0 (BUSY=0) before reading results. Math commands complete in under 1 µs — faster than the Z80 can send the operands.

### System (opcodes &00–&05)

| Op | Name | Operands → &FF41 | Result ← &FF42 |
|----|------|-----------------|----------------|
| &00 | NOP | none | none |
| &01 | VERSION | none | 2 bytes — major, minor |
| &02 | ZBUF_CLEAR | none | none — DMA fill with &FFFF |
| &03 | SET_VIEWPORT | x(2) y(2) w(2) h(2) | none |
| &04 | TEX_SIZE | w(1) h(1) | none — must be power of 2 |
| &05 | SET_VGA_MODE | mode(1) | none — see VGA modes table |

### Integer arithmetic (opcodes &10–&13) — hardware multiplier, 1 cycle

| Op | Name | Operands | Result |
|----|------|----------|--------|
| &10 | MUL16 | A(2) B(2) | product(4) |
| &11 | MUL32 | A(4) B(4) | product lo(4) |
| &12 | DIV16 | dividend(2) divisor(2) | quotient(2) remainder(2) |
| &13 | SQRT32 | value(4) | root(2) |

### Trigonometry and fixed-point (opcodes &20–&23)

Angles: 0–255 = 0°–359°. Results: 8.8 fixed-point (signed). 16.16 = 16-bit integer + 16-bit fraction.

| Op | Name | Operands | Result |
|----|------|----------|--------|
| &20 | SIN | angle(1) | sin(2) 8.8 |
| &21 | COS | angle(1) | cos(2) 8.8 |
| &22 | ATAN2 | y(2) x(2) | angle(1) |
| &23 | FP_MUL | A(4) 16.16, B(4) 16.16 | result(4) 16.16 |

### 3D vector and matrix (opcodes &30–&33) — all values 16.16

| Op | Name | Operands | Result |
|----|------|----------|--------|
| &30 | VEC_DOT | A.xyz(12) B.xyz(12) | dot(4) |
| &31 | VEC_CROSS | A.xyz(12) B.xyz(12) | result.xyz(12) |
| &32 | MAT_TRANSFORM | matrix 4×4(64) vertex xyzw(16) | result xyzw(16) |
| &33 | PERSPECTIVE | x y z focal (16 total) | sx sy 1/z (12) |

### 3D rasteriser (opcodes &40–&46) — non-blocking, queues to Core 1

| Op | Name | Operands | Notes |
|----|------|----------|-------|
| &40 | TRI_FLAT | x1 y1 x2 y2 x3 y3 col — 14 bytes | Flat colour, no Z test |
| &41 | TRI_FLAT_Z | x1 y1 z1 … x3 y3 z3 col — 20 bytes | Flat colour + Z-buffer |
| &42 | TRI_TEXTURED | x1 y1 u1 v1 … x3 y3 u3 v3 — 24 bytes | Affine UV, no Z |
| &43 | TRI_TEXTURED_Z | x1 y1 z1 u1 v1 … x3 y3 z3 u3 v3 — 30 bytes | Primary 3D command |
| &44 | SPAN_DRAW | xs xe y u v du dv — 14 bytes | Raw textured scanline |
| &45 | LINE_3D | x1 y1 z1 x2 y2 z2 col — 14 bytes | Z-tested wireframe |
| &46 | BILLBOARD | x y z tw th scale — 12 bytes | Perspective-scaled sprite |

---

## 13. ROM Serving and RAM Banking

### ROM slots we serve

We are the sole ROM provider. ROMDIS is hardwired HIGH — all CPC ROM chips are permanently disabled.

| Slot | Content | Source |
|------|---------|--------|
| 0 | CPC 6128 BASIC ROM | 16 KB in RP2350B flash (XIP, zero SRAM cost) |
| 1–4 | Empty — return &FF | |
| **5** | **Accelerator ROM — RSX + ASM library** | 16 KB in RP2350B flash |
| 6 | Empty — return &FF | |
| 7 | CPC 6128 AMSDOS ROM | 16 KB in RP2350B flash |
| 8–31 | Empty — return &FF | |

**ROM slot 5** was chosen because the CPC 464 scans slots 0–7 at boot (6128 scans 0–15). Slot 5 is within the 464 scan range, away from the reserved 0 (BASIC) and 7 (AMSDOS) positions.

### ROMEN — not needed

ROMEN (CPC expansion pin 42) is normally used by ROM expansion boards to know when to enable their chip. We do not connect it. Instead we track the Gate Array ROM-enable bits by snooping all &7Fxx writes and maintaining `lower_rom_enabled` and `upper_rom_enabled` shadow flags. These are initialised to the CPC power-on state before /RESET is released, so no startup race condition exists.

### Memory read arbitration

```c
void handle_memory_read(uint16_t addr) {

    if (addr < 0x4000) {
        // Lower ROM region
        if (lower_rom_enabled)
            drive_data_bus(cpc6128_basic[addr]);      // BASIC from flash
        else
            drive_data_bus(page_map[0][addr]);        // RAM page 0
        return;
    }

    if (addr >= 0xC000) {
        // Upper ROM region
        uint16_t off = addr - 0xC000;
        if (upper_rom_enabled) {
            switch (current_upper_rom) {
                case 0:  drive_data_bus(cpc6128_basic[off]);  return;
                case 5:  drive_data_bus(accel_rom[off]);      return;
                case 7:  drive_data_bus(cpc6128_amsdos[off]); return;
                default: drive_data_bus(0xFF);                return;
            }
        } else {
            drive_data_bus(page_map[3][off]);         // RAM page 3
        }
        return;
    }

    // &4000–&BFFF — always banked RAM
    drive_data_bus(page_map[addr >> 14][addr & 0x3FFF]);
}
```

### 128 KB RAM banking

Eight 16 KB pages (pages 0–7). Gate Array &C0 command (bits 7:6 = 11) selects one of 8 configurations:

```c
static const uint8_t page_table[8][4] = {
    {0, 1, 2, 3},  // config 0: standard — same as CPC 464
    {0, 1, 2, 7},  // config 1: &C000 → page 7
    {4, 5, 6, 7},  // config 2: all extra pages
    {0, 3, 2, 7},  // config 3
    {0, 4, 2, 3},  // config 4
    {0, 5, 2, 3},  // config 5
    {0, 6, 2, 3},  // config 6
    {0, 7, 2, 3},  // config 7
};

void apply_bank_config(uint8_t config) {
    for (int i = 0; i < 4; i++)
        page_map[i] = ram[page_table[config & 7][i]];
}
```


---

# PART 3 — FIRMWARE

---

## 14. Boot Sequence

```
On /RESET (from CPC or power-on):

  1. Zero all 8 pages of RAM (128 KB total)
  2. Set banking config 0 — standard layout (identical to CPC 464)
  3. Initialise Gate Array shadow:
       mode = 1 (320×200, 4 colours)
       palette = CPC firmware defaults
       lower ROM enabled, upper ROM enabled
  4. Initialise CRTC shadow to firmware defaults:
       R0=63 R1=40 R2=46 R3=&8E R4=38 R6=25 R7=30 R9=7 R12=&30 R13=0
  5. Set PLL_USB for 13.0 MHz pixel clock → 50 Hz CPC native VGA output
  6. Start VGA PIO1 — display mode = CPC NATIVE
  7. RAMDIS and ROMDIS already hardwired HIGH — no action needed
  8. Release /RESET — Z80 begins executing

  Z80 reads from &0000. Gate Array has lower ROM enabled.
  We serve 6128 BASIC from flash. ROMDIS is HIGH — CPC ROM chip disabled.
  Core 1 simultaneously reconstructs and outputs the CPC screen.
  User sees BASIC start screen within one video frame.
```


```svg
<svg width="100%" viewBox="0 0 1800 620" xmlns="http://www.w3.org/2000/svg" style="font-family:sans-serif;background:#1a1a2e">
  <defs>
    <marker id="a" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="#888" stroke-width="1.5" stroke-linecap="round"/>
    </marker>
  </defs>

  <!-- Title -->
  <text x="900" y="28" text-anchor="middle" fill="#aaf" font-size="18" font-weight="bold">Dual-Core Firmware Architecture</text>

  <!-- Z80 CPC -->
  <rect x="20" y="44" width="190" height="560" rx="8" fill="#2d2d44" stroke="#6c6caa" stroke-width="1.5"/>
  <text x="115" y="72" text-anchor="middle" fill="#aac" font-size="16" font-weight="bold">CPC Z80</text>
  <text x="115" y="90" text-anchor="middle" fill="#667" font-size="12">4 MHz</text>
  <line x1="30" y1="100" x2="200" y2="100" stroke="#333" stroke-width="0.5"/>
  <text x="115" y="120" text-anchor="middle" fill="#9cf" font-size="12">RAM read/write</text>
  <text x="115" y="140" text-anchor="middle" fill="#fc9" font-size="12">OUT &amp;FF60 (VRAM)</text>
  <text x="115" y="160" text-anchor="middle" fill="#fc9" font-size="12">OUT &amp;FF63 (reg)</text>
  <text x="115" y="180" text-anchor="middle" fill="#fc9" font-size="12">OUT &amp;FF64 (sel)</text>
  <text x="115" y="210" text-anchor="middle" fill="#aaf" font-size="12">OUT &amp;FF40 (cmd)</text>
  <text x="115" y="230" text-anchor="middle" fill="#aaf" font-size="12">OUT &amp;FF41 (data)</text>
  <text x="115" y="260" text-anchor="middle" fill="#9fb" font-size="12">OUT &amp;7Fxx (GA)</text>
  <text x="115" y="280" text-anchor="middle" fill="#9fb" font-size="12">OUT &amp;BCxx (CRTC)</text>
  <text x="115" y="300" text-anchor="middle" fill="#9fb" font-size="12">OUT &amp;DFxx (ROM sel)</text>
  <text x="115" y="330" text-anchor="middle" fill="#f99" font-size="12">IN &amp;FF43 (status)</text>
  <text x="115" y="350" text-anchor="middle" fill="#f99" font-size="12">IN &amp;FF64 (V9990 st)</text>

  <!-- Core 0 -->
  <rect x="250" y="44" width="420" height="560" rx="8" fill="#2a1a1a" stroke="#cc6633" stroke-width="1.5"/>
  <text x="460" y="72" text-anchor="middle" fill="#fa8" font-size="16" font-weight="bold">Core 0 — Bus Handler</text>
  <text x="460" y="90" text-anchor="middle" fill="#665" font-size="12">interrupt-driven · deterministic latency · never blocks</text>
  <line x1="260" y1="100" x2="660" y2="100" stroke="#333" stroke-width="0.5"/>

  <rect x="268" y="110" width="384" height="60" rx="5" fill="#1a1a2a" stroke="#3399cc" stroke-width="1"/>
  <text x="460" y="134" text-anchor="middle" fill="#6cf" font-size="13" font-weight="bold">PIO SM0 — unified write snooper</text>
  <text x="460" y="154" text-anchor="middle" fill="#446" font-size="12">fires on /WR↓  ·  in pins,32 captures full bus state  ·  pushes to FIFO</text>

  <rect x="268" y="180" width="384" height="60" rx="5" fill="#1a1a2a" stroke="#3399cc" stroke-width="1"/>
  <text x="460" y="204" text-anchor="middle" fill="#6cf" font-size="13" font-weight="bold">PIO SM1 — memory read handler</text>
  <text x="460" y="224" text-anchor="middle" fill="#446" font-size="12">/MREQ=0 + /WR=1  →  IRQ Core 0  →  drive bus  (~73 ns total)</text>

  <rect x="268" y="250" width="384" height="80" rx="5" fill="#1a2a1a" stroke="#336633" stroke-width="1"/>
  <text x="460" y="274" text-anchor="middle" fill="#8f8" font-size="13" font-weight="bold">Write decoder</text>
  <text x="460" y="294" text-anchor="middle" fill="#446" font-size="12">/MREQ=0  →  RAM write to page_map[addr>>14]</text>
  <text x="460" y="312" text-anchor="middle" fill="#446" font-size="12">/MREQ=1  →  I/O write  →  check B register</text>

  <rect x="268" y="340" width="384" height="100" rx="5" fill="#1a2a1a" stroke="#336633" stroke-width="1"/>
  <text x="460" y="364" text-anchor="middle" fill="#8f8" font-size="13" font-weight="bold">Port router</text>
  <text x="460" y="384" text-anchor="middle" fill="#446" font-size="12">B=&amp;7F  → GA shadow update (palette, mode, banking)</text>
  <text x="460" y="402" text-anchor="middle" fill="#446" font-size="12">B=&amp;BC  → CRTC shadow update (all 18 regs tracked)</text>
  <text x="460" y="420" text-anchor="middle" fill="#446" font-size="12">B=&amp;FF &amp;60–6F  →  V9990 handler</text>
  <text x="460" y="438" text-anchor="middle" fill="#446" font-size="12">B=&amp;FF &amp;40–4F  →  coprocessor command ring push</text>

  <rect x="268" y="450" width="384" height="60" rx="5" fill="#2a1a1a" stroke="#993333" stroke-width="1"/>
  <text x="460" y="474" text-anchor="middle" fill="#f88" font-size="13" font-weight="bold">ROM/RAM arbiter</text>
  <text x="460" y="494" text-anchor="middle" fill="#446" font-size="12">serves ROM slots 0/5/7 from flash XIP  ·  banked RAM from SRAM</text>

  <rect x="268" y="520" width="384" height="66" rx="5" fill="#2a1a2a" stroke="#996699" stroke-width="1"/>
  <text x="460" y="544" text-anchor="middle" fill="#c9c" font-size="13" font-weight="bold">/INT generation</text>
  <text x="460" y="564" text-anchor="middle" fill="#446" font-size="12">fires every 52 CRTC scanlines  ·  INTACK modelled ≈1 µs after</text>

  <!-- Command ring -->
  <rect x="716" y="280" width="120" height="200" rx="8" fill="#2a2a00" stroke="#aaaa33" stroke-width="1.5"/>
  <text x="776" y="308" text-anchor="middle" fill="#ff9" font-size="13" font-weight="bold">cmd ring</text>
  <text x="776" y="328" text-anchor="middle" fill="#775" font-size="12">512 bytes</text>
  <text x="776" y="348" text-anchor="middle" fill="#775" font-size="12">in SRAM</text>
  <text x="776" y="376" text-anchor="middle" fill="#775" font-size="12">lockless</text>
  <text x="776" y="396" text-anchor="middle" fill="#775" font-size="12">Core 0</text>
  <text x="776" y="412" text-anchor="middle" fill="#775" font-size="12">writes</text>
  <text x="776" y="436" text-anchor="middle" fill="#775" font-size="12">Core 1</text>
  <text x="776" y="452" text-anchor="middle" fill="#775" font-size="12">reads</text>

  <!-- Core 1 -->
  <rect x="882" y="44" width="440" height="560" rx="8" fill="#1a2a2a" stroke="#33aacc" stroke-width="1.5"/>
  <text x="1102" y="72" text-anchor="middle" fill="#6cc" font-size="16" font-weight="bold">Core 1 — Multiplex Engine</text>
  <text x="1102" y="90" text-anchor="middle" fill="#456" font-size="12">three priority levels · DMA-driven scanline output</text>
  <line x1="892" y1="100" x2="1312" y2="100" stroke="#333" stroke-width="0.5"/>

  <rect x="900" y="110" width="404" height="110" rx="5" fill="#1a1a3a" stroke="#3399cc" stroke-width="1"/>
  <text x="1102" y="134" text-anchor="middle" fill="#6cf" font-size="13" font-weight="bold">Priority 1 — VGA Scanline Renderer (preemptive)</text>
  <text x="1102" y="154" text-anchor="middle" fill="#446" font-size="12">DMA IRQ fires each line  ·  runs CRTC state machine</text>
  <text x="1102" y="172" text-anchor="middle" fill="#446" font-size="12">applies GA events at exact character position</text>
  <text x="1102" y="192" text-anchor="middle" fill="#446" font-size="12">CPC native: full overscan  ·  V9990: VRAM decode  ·  direct: framebuffer</text>
  <text x="1102" y="212" text-anchor="middle" fill="#446" font-size="12">writes to double-buffered line buffer  →  DMA  →  PIO RGB555</text>

  <rect x="900" y="232" width="404" height="90" rx="5" fill="#1a2a1a" stroke="#336633" stroke-width="1"/>
  <text x="1102" y="256" text-anchor="middle" fill="#8f8" font-size="13" font-weight="bold">Priority 2 — Math Coprocessor</text>
  <text x="1102" y="276" text-anchor="middle" fill="#446" font-size="12">pops from command ring  ·  executes  ·  writes result_buf[] in SRAM</text>
  <text x="1102" y="296" text-anchor="middle" fill="#446" font-size="12">MUL16: 1 cycle  ·  MAT_TRANSFORM: ~200 cycles  ·  all &lt;1 µs</text>
  <text x="1102" y="314" text-anchor="middle" fill="#446" font-size="12">effectively instantaneous from Z80 perspective</text>

  <rect x="900" y="334" width="404" height="110" rx="5" fill="#2a1a1a" stroke="#cc6633" stroke-width="1"/>
  <text x="1102" y="358" text-anchor="middle" fill="#fa8" font-size="13" font-weight="bold">Priority 3 — 3D Triangle Rasteriser (background)</text>
  <text x="1102" y="378" text-anchor="middle" fill="#446" font-size="12">TRI_TEXTURED_Z: reads texture from PSRAM</text>
  <text x="1102" y="396" text-anchor="middle" fill="#446" font-size="12">interpolates UV  ·  Z-buffer tests  ·  writes pixels to PSRAM VRAM</text>
  <text x="1102" y="414" text-anchor="middle" fill="#446" font-size="12">preempted by scanline IRQ  ·  resumes after</text>
  <text x="1102" y="434" text-anchor="middle" fill="#446" font-size="12">vblank burst: ~142,000 pixels  ·  active gaps: ~995,000 pixels/frame</text>

  <rect x="900" y="456" width="404" height="66" rx="5" fill="#1a2a1a" stroke="#33cc99" stroke-width="1"/>
  <text x="1102" y="480" text-anchor="middle" fill="#6fc" font-size="13" font-weight="bold">PIO1 — VGA pixel output</text>
  <text x="1102" y="500" text-anchor="middle" fill="#446" font-size="12">DMA feeds line buffer  →  SM1 out pins,15  →  GPIO32–46</text>
  <text x="1102" y="518" text-anchor="middle" fill="#446" font-size="12">SM0 generates HSYNC/VSYNC  ·  PLL_USB sets exact pixel clock</text>

  <!-- PSRAM box -->
  <rect x="1370" y="44" width="220" height="560" rx="8" fill="#1a2a1a" stroke="#339966" stroke-width="1.5"/>
  <text x="1480" y="72" text-anchor="middle" fill="#6c9" font-size="14" font-weight="bold">8 MB PSRAM</text>
  <text x="1480" y="90" text-anchor="middle" fill="#456" font-size="11">QSPI1</text>
  <rect x="1382" y="104" width="196" height="50" rx="4" fill="#1a3a2a" stroke="#336633"/>
  <text x="1480" y="126" text-anchor="middle" fill="#8fb" font-size="12">V9990 VRAM</text>
  <text x="1480" y="144" text-anchor="middle" fill="#567" font-size="11">512 KB</text>
  <rect x="1382" y="162" width="196" height="50" rx="4" fill="#1a3a2a" stroke="#336633"/>
  <text x="1480" y="184" text-anchor="middle" fill="#8fb" font-size="12">VGA framebuffer</text>
  <text x="1480" y="202" text-anchor="middle" fill="#567" font-size="11">2 MB</text>
  <rect x="1382" y="220" width="196" height="50" rx="4" fill="#1a3a2a" stroke="#336633"/>
  <text x="1480" y="242" text-anchor="middle" fill="#8fb" font-size="12">Z-buffer</text>
  <text x="1480" y="260" text-anchor="middle" fill="#567" font-size="11">512 KB</text>
  <rect x="1382" y="278" width="196" height="50" rx="4" fill="#1a3a2a" stroke="#336633"/>
  <text x="1480" y="300" text-anchor="middle" fill="#8fb" font-size="12">Texture RAM</text>
  <text x="1480" y="318" text-anchor="middle" fill="#567" font-size="11">2 MB</text>
  <rect x="1382" y="336" width="196" height="50" rx="4" fill="#1a3a2a" stroke="#336633"/>
  <text x="1480" y="358" text-anchor="middle" fill="#8fb" font-size="12">Backing store</text>
  <text x="1480" y="376" text-anchor="middle" fill="#567" font-size="11">3 MB</text>

  <!-- Arrows: CPC → Core 0 -->
  <line x1="210" y1="200" x2="250" y2="200" stroke="#888" stroke-width="1.5" marker-end="url(#a)"/>
  <line x1="210" y1="280" x2="250" y2="280" stroke="#888" stroke-width="1.5" marker-end="url(#a)"/>
  <line x1="210" y1="340" x2="250" y2="340" stroke="#888" stroke-width="1.5" marker-end="url(#a)"/>

  <!-- Core 0 → ring -->
  <line x1="670" y1="380" x2="716" y2="380" stroke="#aaaa33" stroke-width="2" marker-end="url(#a)"/>
  <!-- Ring → Core 1 -->
  <line x1="836" y1="380" x2="882" y2="380" stroke="#aaaa33" stroke-width="2" marker-end="url(#a)"/>

  <!-- Core 1 → PSRAM -->
  <line x1="1322" y1="300" x2="1370" y2="300" stroke="#339966" stroke-width="2" marker-end="url(#a)" marker-start="url(#a)"/>
  <text x="1346" y="290" text-anchor="middle" fill="#6a9" font-size="10">QSPI</text>

  <!-- Status feedback: Core 1 → Core 0 -->
  <line x1="882" y1="540" x2="670" y2="540" stroke="#996699" stroke-width="1.5" stroke-dasharray="5 4" marker-end="url(#a)"/>
  <text x="776" y="558" text-anchor="middle" fill="#a9a" font-size="10">scanline count · /INT trigger</text>
</svg>
```


---

## 15. Firmware Architecture

### Core 0 — real-time bus handler

```
PIO SM0 — unified write snooper
  Fires on every /WR falling edge
  in pins,32 captures entire bus state in one instruction
  Pushes 32-bit word to FIFO

Core 0 main loop — pops FIFO, routes by address and /MREQ:

  /MREQ=0, /WR=0   → RAM write
                      emulated_ram page_map[addr>>14][addr&0x3FFF] = data

  /MREQ=1, /WR=0   → I/O write — check B register (high byte):
    B=&7F           → Gate Array write
                        mode/palette/ROM-enable/bank config shadow update
    B=&BC–&BF       → CRTC write
                        update crtc_regs[], notify CRTC state machine
    B=&DF           → ROM select
                        update current_upper_rom (0–31)
    B=&FF, C=&60–6F → V9990 write
                        VRAM poke, palette, register, system control
    B=&FF, C=&40–4F → Coprocessor write
                        push to command ring buffer

PIO SM1 — memory read handler
  Detects /MREQ=0 + /WR=1 → IRQ Core 0
  Core 0 ISR calls handle_memory_read(addr)
  Drives byte onto D0–D7 via GPIO + 74LVC245
  Total latency ~73 ns — well within Z80 timing

Status outputs:
  Pre-loads &FF43 (copro STATUS) and &FF64 (V9990 STATUS) into PIO TX FIFOs
  Drives /INT to CPC every 52 CRTC scanlines (GA interrupt counter)
```

### Core 1 — three-priority multiplex engine

```
Priority 1 — VGA scanline renderer (DMA IRQ, preempts everything)
  Fires every line period based on current VGA mode
  Reads VRAM from PSRAM, decodes V9990 pixel format
  For CPC native: runs CRTC state machine, applies GA events, renders full frame
  Composites sprites, writes to double-buffered SRAM line buffer
  DMA feeds line buffer to PIO1 pixel serialiser

Priority 2 — Math coprocessor (runs in gaps between scanline renders)
  Pops commands from ring buffer, executes, writes to result_buf[] in SRAM
  All operations complete in < 1 µs — done before Z80 can poll STATUS

Priority 3 — 3D triangle rasteriser (background + full vblank burst)
  TRI_TEXTURED_Z: reads texture from PSRAM, Z-tests, writes pixels to PSRAM VRAM
  Preempted by scanline IRQ, resumes after
  Vblank period (no scanline preemption) used for burst rasterisation
```

### Gate Array snooping — all writes decoded

Every write to &7Fxx is decoded by Core 0 immediately:

```c
void handle_gate_array_write(uint8_t data) {
    uint8_t sel = data >> 6;
    switch (sel) {
        case 0:  // ink select
            ga_ink_select = data & 0x1F;
            break;
        case 1:  // ink colour
            ga_palette[ga_ink_select] = data & 0x1F;
            // Tag with CRTC position for scanline-accurate reconstruction
            push_ga_event(data, crtc.vcc, crtc.raster, crtc.hcc);
            break;
        case 2:  // ROM config + screen mode
            lower_rom_enabled = !(data & 0x04);
            upper_rom_enabled = !(data & 0x08);
            ga_mode = data & 0x03;
            push_ga_event(data, crtc.vcc, crtc.raster, crtc.hcc);
            break;
        case 3:  // RAM banking
            apply_bank_config(data & 0x07);
            break;
    }
}
```

### CRTC snooping — all 18 registers tracked

```c
void handle_crtc_write(uint8_t port_lo, uint8_t data) {
    if ((port_lo & 0x03) == 0x00)
        crtc_reg_select = data & 0x1F;
    else if ((port_lo & 0x03) == 0x01) {
        crtc_regs[crtc_reg_select] = data;
        // Notify Core 1 state machine which register changed
        crtc_reg_dirty |= (1u << crtc_reg_select);
    }
}
```

---

## 16. VGA Output — Dual PLL Architecture

### Why two PLLs

- **PLL_SYS — fixed at 294 MHz** (12 × 49 ÷ 2 — exact). Drives CPU cores, SRAM, PIO0 (bus handler). Never reprogrammed.
- **PLL_USB — reprogrammed per mode**. Drives PIO1 only (VGA). Changing it has zero effect on the bus handler or CPU.

Mode switches: Core 1 stops PIO1, reprograms PLL_USB (~200 µs lock = 1–2 blank frames), reloads sync timing, restarts PIO1. Z80 bus keeps running throughout.

### PLL_USB configurations — exact pixel clocks

| Pixel clock | Modes | fbdiv | pd1 | pd2 | Actual | Error |
|-------------|-------|-------|-----|-----|--------|-------|
| 13.0 MHz | CPC native 50 Hz | 65 | 3 | 5 | 13.000 MHz | exact |
| 25.2 MHz | 640×480@60, 640×400@70 | 63 | 5 | 6 | 25.200 MHz | 0.099% |
| 31.5 MHz | 640×480@75 | 63 | 4 | 6 | 31.500 MHz | exact |
| 40.0 MHz | 800×600@60 | 70 | 3 | 7 | 40.000 MHz | exact |
| 49.5 MHz | 800×600@75 | 66 | 4 | 4 | 49.500 MHz | exact |
| 50.0 MHz | 800×600@72 | 75 | 3 | 6 | 50.000 MHz | exact |
| 65.0 MHz | 1024×768@60 | 65 | 2 | 6 | 65.000 MHz | exact |
| 74.25 MHz | 1280×720@60 | 99 | 4 | 4 | 74.250 MHz | exact |
| 75.0 MHz | 1024×768@70 | 75 | 2 | 6 | 75.000 MHz | exact |
| 78.75 MHz | 1024×768@75 | 105 | 4 | 4 | 78.750 MHz | exact |
| 108.0 MHz | 1280×1024@60 | 63 | 1 | 7 | 108.000 MHz | exact |
| 148.5 MHz | 1920×1080@60 | 99 | 2 | 4 | 148.500 MHz | exact |

### All VGA modes (SET_VGA_MODE command &05)

Mode 0 = CPC native reconstruction (50 Hz). Modes 1–33 = direct VGA framebuffer (RGB555, 2 bytes/pixel, all fit within 2 MB PSRAM framebuffer). ★ = recommended double-scan mode for that output resolution.

| Mode | Render res | Output | FB size | Method |
|------|-----------|--------|---------|--------|
| 0 | CPC native | 50 Hz reconstruct | — | CRTC state machine |
| 1 | 160×120 | 640×480@60 | 38 KB | 4×4 upscale |
| 2 | 320×120 | 640×480@60 | 75 KB | 2×h 4×v |
| 3 | 320×240 | 640×480@60 | 150 KB | 2×2 upscale |
| 4 | 640×240 | 640×480@60 | 300 KB | double-scan |
| 5 | 640×480 | 640×480@60 | 600 KB | native |
| 6 | 320×200 | 640×400@70 | 125 KB | 2×2 upscale |
| 7 | 640×200 | 640×400@70 | 250 KB | double-scan |
| 8 | 640×400 | 640×400@70 | 500 KB | native |
| 9 | 320×240 | 640×480@75 | 150 KB | 2×2 upscale |
| 10 | 640×240 | 640×480@75 | 300 KB | double-scan |
| 11 | 640×480 | 640×480@75 | 600 KB | native 75 Hz |
| 12 | 200×150 | 800×600@60 | 59 KB | 4×4 upscale |
| 13 | 400×300 | 800×600@60 | 234 KB | 2×2 upscale |
| 14 | 800×300 | 800×600@60 | 469 KB | double-scan |
| 15 | 800×600 | 800×600@60 | 938 KB | native (buf) |
| 16 | 800×300 | 800×600@72 | 469 KB | double-scan |
| 17 | 800×600 | 800×600@72 | 938 KB | native (buf) |
| 18 | 256×192 | 1024×768@60 | 96 KB | 4×4 upscale |
| 19 | 512×384 | 1024×768@60 | 384 KB | 2×2 upscale |
| 20★ | 1024×384 | 1024×768@60 | 768 KB | double-scan |
| 21 | 1024×768 | 1024×768@60 | 1536 KB | native (buf) |
| 22★ | 1024×384 | 1024×768@70 | 768 KB | double-scan |
| 23 | 1024×768 | 1024×768@70 | 1536 KB | native (buf) |
| 24 | 320×180 | 1280×720@60 | 112 KB | 4×4 upscale |
| 25 | 640×360 | 1280×720@60 | 450 KB | 2×2 upscale |
| 26★ | 1280×360 | 1280×720@60 | 900 KB | double-scan |
| 27 | 1280×720 | 1280×720@60 | 1800 KB | native (buf) |
| 28 | 320×256 | 1280×1024@60 | 160 KB | 4×4 upscale |
| 29 | 640×512 | 1280×1024@60 | 640 KB | 2×2 upscale |
| 30★ | 1280×512 | 1280×1024@60 | 1280 KB | double-scan (buf) |
| 31 | 480×270 | 1920×1080@60 | 253 KB | 4×4 upscale |
| 32 | 960×270 | 1920×1080@60 | 506 KB | 2×h 4×v |
| 33★ | 1920×270 | 1920×1080@60 | 1013 KB | 4×v scan (buf) |

(buf) = requires 2-line-ahead SRAM scanline buffer — already in firmware design.

---

## 17. CPC Native Display — Cycle-Accurate Reconstruction

### 17.1 The CRTC state machine

Core 1 runs a complete software model of the 6845 CRTC, advancing every 1 µs (character clock = 16 MHz ÷ 16). This is the foundation for all advanced display effects.

```c
typedef struct {
    uint8_t  hcc;           // horizontal character counter (0..R0)
    uint8_t  vcc;           // vertical character counter (0..R4)
    uint8_t  raster;        // raster line within character row (0..R9)
    bool     hsync, vsync;  // sync states
    bool     display_en;    // true = active display area
    uint16_t ma;            // CRTC memory address (from R12/R13 + counters)
    uint8_t  ga_int_count;  // GA interrupt counter (0..51)
    uint8_t  regs[18];      // shadow of all 18 CRTC registers
} crtc_state_t;

void crtc_tick(crtc_state_t *s) {
    s->hcc++;
    if (s->hcc > s->regs[0]) {           // end of line
        s->hcc = 0;
        s->raster++;
        s->ga_int_count++;
        if (s->ga_int_count >= 52) {
            s->ga_int_count = 0;
            assert_int_to_cpc();          // drive /INT low
        }
        if (s->raster > s->regs[9]) {    // end of character row
            s->raster = 0;
            s->vcc++;
            if (s->vcc > s->regs[4])
                s->vcc = 0;
        }
    }
    update_display_enable(s);
    update_sync_outputs(s);
    advance_ma(s);
}
```

### 17.2 Gate Array event tagging

Every GA write captured by Core 0 is tagged with the current CRTC state, giving **character-clock (1 µs) resolution** for when changes take effect:

```c
typedef struct {
    uint8_t ga_data;
    uint8_t scanline;   // vcc * (R9+1) + raster
    uint8_t hcc;        // horizontal char position
} ga_event_t;
```

Core 1 dequeues these events and applies them at the exact position during scanline rendering — raster colour bars, mid-line mode switches, mid-frame palette changes all handled correctly.

### 17.3 INTACK approximation

Without /IORQ and /M1 we cannot detect the Z80 interrupt acknowledge cycle directly. We model INTACK as occurring **1 µs after /INT assertion** and reset the GA interrupt counter at that point. This is accurate to ±1 scanline for the vast majority of demos and games. The only failure case is code that deliberately delays INTACK to shift the counter phase — an advanced technique used by a small number of highly optimised demos.

### 17.4 Overscan and full-frame rendering

We render the full CRTC addressable area — not just the standard 200 lines. Maximum overscan: ~768 pixels wide, ~288 lines tall. The VGA output in CPC native mode uses 832×312 total pixels at 50 Hz (13.0 MHz pixel clock, exact from PLL_USB) to cover this.

### 17.5 Screen memory addressing

The CPC screen is not a linear framebuffer. Character rows are stored with **2048 bytes between consecutive scan lines** within the same character row:

```
addr = screen_base + (cpc_y/8)*80 + (cpc_y%8)*2048 + byte_x
```

Mode 1 pixel decode (4 pixels per byte):
```
pixel n = ((byte >> (7-n)) & 1) << 1 | ((byte >> (3-n)) & 1)   n = 0..3
```

### 17.6 Advanced display technique support

| Technique | Supported | Notes |
|-----------|-----------|-------|
| Overscan | ✓ Full | Up to 768×288 pixels |
| Raster bars — border colour changes | ✓ Full | 1 µs resolution |
| Mid-frame palette changes | ✓ Full | Tagged with CRTC position |
| Mid-scanline mode switch | ✓ Full | Sub-character resolution |
| Split screen via R12/R13 | ✓ Full | Applied immediately |
| Rapture — R2 mid-frame changes | ✓ Full | Per-line HSYNC position |
| Line doubling via R9 | ✓ Full | Applied immediately |
| Interrupt-timed effects | ✓ Approx | ±1 scanline — INTACK modelled |
| 50 Hz native output | ✓ Full | No frame skipping or judder |

---

## 18. PSRAM Layout (8 MB)

| Region | Address | Size | Contents |
|--------|---------|------|---------|
| V9990 VRAM | 0x000000–0x07FFFF | 512 KB | Pattern planes, sprite table, palette |
| VGA framebuffer | 0x080000–0x27FFFF | 2 MB | Direct VGA modes (RGB555) |
| Z-buffer | 0x280000–0x2FFFFF | 512 KB | 16-bit depth, 2 bytes/pixel |
| Texture RAM | 0x300000–0x4FFFFF | 2 MB | Game and app textures |
| Backing store | 0x500000–0x7FFFFF | 3 MB | Window save/restore, sprite cache |

### Frame performance budget (640×480 @ 60 Hz)

| Period | Duration | Core 1 cycles | Used for |
|--------|----------|--------------|---------|
| Per scanline — render | ~11 µs | ~3,240 | Scanline decode + CRTC advance |
| Per scanline — free | ~21 µs | ~6,174 | Math commands + ~2,058 triangle pixels |
| Vblank (45 lines) | 1.43 ms | ~420,420 | Burst rasterisation |
| **Total 3D pixels/frame** | | **~1,108,000** | Real-time 3D at 60 fps |

Z80 at 4 MHz: ~66,700 cycles/frame. Core 1 at 294 MHz: ~4,900,000 cycles/frame. **Core 1 has 73× more compute than the Z80.**


---

# PART 4 — SOFTWARE

---

## 19. Accelerator ROM — Upper ROM Slot 5

### How CPC upper ROMs work

BASIC scans slots 0–7 on boot (CPC 464) or 0–15 (6128), finds foreground ROMs (type byte = 1), calls their init routines, and registers their RSX commands. **This is automatic — no LOAD or CALL needed.**

We serve slot 5. It is within the 464 scan range, away from reserved slots 0 (BASIC) and 7 (AMSDOS).

### ROM memory map (&C000–&FFFF, 16 KB, served from flash XIP)

```
&C000  ROM type (&01), mark (&CB), version bytes
&C003  JP RESET_HANDLER
&C006  RETI  (background entry — not used for type 1)
&C009  JP APP_INIT
&C00C  Product name: "ACCEL V1", 0

&C010  ┌─ Fixed API jump table — 26 × JP nnnn (78 bytes) ─────────┐
       │  These addresses NEVER move between ROM versions.          │
       │  User code can always CALL &C010 etc. safely.             │
&C060  └────────────────────────────────────────────────────────────┘

&C080  RSX name table  (|ACC_MODE |ACC_FILL |ACC_BLIT ... |ACC_INFO)
&C200  RSX handler address table  (2 bytes per handler)
&C300  RSX handler code
&C800  ASM library implementation  (jump table targets)
&CC00  APP_INIT — card detect, RSX register, default PSRAM addresses
&CD00  CPC 27-colour RGB555 lookup table  +  Mode 1 pixel mask table
&CE00  VGA mode parameter table
&CF00  Reserved
&FFFF
```

### Fixed API jump table (&C010–&C05D)

Every entry is `JP nnnn` (3 bytes). Addresses frozen forever.

```
CALL &C010  ACCEL_WAIT          CALL &C034  MAT_TRANSFORM
CALL &C013  ACCEL_CMD_START     CALL &C037  PERSPECTIVE
CALL &C016  ACCEL_SEND_WORD     CALL &C03A  GFX_FILL_RECT
CALL &C019  ACCEL_READ_WORD     CALL &C03D  GFX_BLIT
CALL &C01C  STREAM_N            CALL &C040  GFX_SCROLL
CALL &C01F  MUL16               CALL &C043  GFX_LINE
CALL &C022  DIV16               CALL &C046  GFX_TRI_Z
CALL &C025  SQRT32              CALL &C049  GFX_CLS
CALL &C028  GET_SIN             CALL &C04C  SET_VGA_MODE
CALL &C02B  GET_COS             CALL &C04F  SET_TEX_ADDR
CALL &C02E  ATAN2               CALL &C052  SET_ZBUF_ADDR
CALL &C031  FP_MUL              CALL &C055  WAIT_VSYNC
                                CALL &C058  V9990_WRITE_VRAM
                                CALL &C05B  V9990_SET_WRITE_ADDR
```

### APP_INIT — called automatically by BASIC firmware at every boot

```asm
APP_INIT:
    LD B, &FF : LD C, &43 : IN A,(C)   ; read copro STATUS
    CP &FF : JR Z, .no_card            ; &FF = bus floating, no card

    LD A, &01 : CALL ACCEL_CMD_START   ; VERSION command
    CALL ACCEL_WAIT
    CALL ACCEL_READ_WORD               ; HL = major.minor
    LD (ROM_VERSION), HL

    LD HL, RSX_NAMES : LD DE, RSX_TABLE
    CALL &BCCB                         ; KL_LOG_EXT — register RSX with BASIC

    LD HL, &0000 : LD A, &28           ; Z-buffer at PSRAM &280000
    CALL SET_ZBUF_ADDR
    LD HL, &0000 : LD A, &30           ; textures at PSRAM &300000
    CALL SET_TEX_ADDR

    LD A, 0 : CALL SET_VGA_MODE        ; mode 0 = CPC native 50 Hz
    RET

.no_card:
    LD HL, MSG : .pr: LD A,(HL) : INC HL : OR A : RET Z : CALL &BB5A : JR .pr
MSG:         DEFB "ACCEL: card not found",13,10,0
ROM_VERSION: DEFW 0
```

---

## 20. BASIC RSX Commands

Registered automatically at every boot. No setup required.

| Command | Parameters | Action |
|---------|-----------|--------|
| `\|ACC_MODE,m` | m = mode ID | Switch VGA output mode (0=CPC native, 5=640×480, etc.) |
| `\|ACC_FILL,x,y,w,h,col` | col = RGB555 | Fill rectangle |
| `\|ACC_BLIT,sx,sy,dx,dy,w,h` | — | Copy rectangle |
| `\|ACC_LINE,x1,y1,x2,y2,col` | — | Draw Bresenham line |
| `\|ACC_SCROLL,x,y,w,h,dx,dy` | dx/dy signed | Scroll rectangle |
| `\|ACC_CLS,col` | — | Clear framebuffer |
| `\|ACC_MUL,a,b,@r` | 16-bit ints | r = a × b (BASIC float) |
| `\|ACC_DIV,a,b,@q,@r` | — | q = quotient, r = remainder |
| `\|ACC_SQRT,vlo,vhi,@r` | 32-bit value | r = integer square root |
| `\|ACC_SIN,ang,@r` | ang = 0–255 | r = sin(ang) × 256, signed |
| `\|ACC_COS,ang,@r` | ang = 0–255 | r = cos(ang) × 256, signed |
| `\|ACC_VSYNC` | — | Wait for vertical blank |
| `\|ACC_VWRITE,vaddr,caddr,len` | — | Copy CPC RAM to V9990 VRAM |
| `\|ACC_VMODE,m` | — | Switch V9990 screen mode |
| `\|ACC_INFO` | — | Print card version to screen |

### BASIC examples

```basic
10 REM RSX available from boot — no LOAD needed
20 |ACC_MODE,5               : REM 640x480 direct VGA
30 |ACC_CLS,0                : REM clear to black
40 FOR C=0 TO 31
50   |ACC_FILL,C*20,100,18,200,C+(C*32)+(C*1024)
60 NEXT C                    : REM colour sweep

70 |ACC_MUL,999,999,@R       : PRINT R    : REM 998001
80 |ACC_SQRT,2000,0,@R       : PRINT R    : REM 44

90 FOR A=0 TO 255
100  |ACC_SIN,A,@S : |ACC_COS,A,@CO
110  X=320+INT(S*100/256) : Y=240+INT(CO*100/256)
120  |ACC_FILL,X,Y,2,2,31744
130 NEXT A                   : REM sine circle

140 |ACC_MODE,0              : REM back to CPC native 50 Hz
```

---

## 21. Z80 ASM Library

Relocatable. No absolute addresses. Called via the fixed jump table at &C010.

### Port constants

```asm
DEV          EQU &FF    ; B register stays &FF throughout
COPRO_CMD    EQU &40    ; W: command opcode
COPRO_DATA   EQU &41    ; W: operand bytes
COPRO_RESULT EQU &42    ; R: result bytes
COPRO_STATUS EQU &43    ; R: BUSY=b0 READY=b1 OVF=b2 DIV0=b3 QFULL=b7
COPRO_ZBUF0  EQU &44    ; W: Z-buf PSRAM addr byte 0 (LSB)
COPRO_ZBUF1  EQU &45
COPRO_ZBUF2  EQU &46    ; W: Z-buf PSRAM addr byte 2 (MSB)
COPRO_TEX0   EQU &47    ; W: texture PSRAM addr byte 0
COPRO_TEX1   EQU &48
COPRO_TEX2   EQU &49
V9990_VRAM   EQU &60    ; R/W: VRAM data, auto-increment
V9990_PAL    EQU &61    ; W: palette (RGB333)
V9990_RDATA  EQU &63    ; R/W: register data
V9990_RSEL   EQU &64    ; W: register select / R: status
V9990_CMD    EQU &65    ; W: blitter command data
V9990_SYS    EQU &67    ; W: system control
V9990_OUT    EQU &6F    ; W: output/IRQ control
```

### Core utility routines (CALL via jump table)

```asm
; ACCEL_WAIT  (CALL &C010) — poll until BUSY=0
; In: nothing  Out: nothing  Destroys: A BC
ACCEL_WAIT:
    LD B,DEV : LD C,COPRO_STATUS
.w: IN A,(C) : BIT 0,A : JR NZ,.w : RET

; ACCEL_CMD_START  (CALL &C013) — send opcode, set BC=&FF41
; In: A=opcode  Out: BC=&FF41  Destroys: BC
ACCEL_CMD_START:
    LD B,DEV : LD C,COPRO_CMD : OUT (C),A : LD C,COPRO_DATA : RET

; ACCEL_SEND_WORD  (CALL &C016) — send HL as 16-bit little-endian
; In: HL=word, BC=&FF41  Destroys: A
ACCEL_SEND_WORD:
    LD A,L : OUT (C),A : LD A,H : OUT (C),A : RET

; ACCEL_READ_WORD  (CALL &C019) — read 16-bit result into HL
; In: nothing  Out: HL=word  Destroys: A BC
ACCEL_READ_WORD:
    LD B,DEV : LD C,COPRO_RESULT
    IN A,(C) : LD L,A : IN A,(C) : LD H,A : RET

; STREAM_N  (CALL &C01C) — send D bytes from (HL) to port BC
; In: D=count HL=src BC=&FF41  Destroys: A D HL
; IMPORTANT: uses D as counter — never B — preserves device select
STREAM_N:
    LD A,(HL) : OUT (C),A : INC HL : DEC D : JR NZ,STREAM_N : RET
```

### Math routines (CALL via jump table)

```asm
; MUL16  (CALL &C01F) — In: HL=A DE=B  Out: HL=lo DE=hi
MUL16:
    CALL ACCEL_WAIT : LD A,&10 : CALL ACCEL_CMD_START
    CALL ACCEL_SEND_WORD : EX DE,HL : CALL ACCEL_SEND_WORD
    CALL ACCEL_WAIT
    CALL ACCEL_READ_WORD : PUSH HL
    CALL ACCEL_READ_WORD : EX DE,HL : POP HL : RET

; DIV16  (CALL &C022) — In: HL=dividend DE=divisor  Out: HL=quotient DE=remainder
DIV16:
    CALL ACCEL_WAIT : LD A,&12 : CALL ACCEL_CMD_START
    CALL ACCEL_SEND_WORD : EX DE,HL : CALL ACCEL_SEND_WORD
    CALL ACCEL_WAIT
    CALL ACCEL_READ_WORD : PUSH HL
    CALL ACCEL_READ_WORD : EX DE,HL : POP HL : RET

; SQRT32  (CALL &C025) — In: HL=value_lo DE=value_hi  Out: HL=root
SQRT32:
    CALL ACCEL_WAIT : LD A,&13 : CALL ACCEL_CMD_START
    CALL ACCEL_SEND_WORD : EX DE,HL : CALL ACCEL_SEND_WORD
    CALL ACCEL_WAIT : CALL ACCEL_READ_WORD : RET

; GET_SIN  (CALL &C028) — In: A=angle(0-255)  Out: HL=sin 8.8
GET_SIN:
    CALL ACCEL_WAIT : PUSH AF : LD A,&20 : CALL ACCEL_CMD_START
    POP AF : OUT (C),A : CALL ACCEL_WAIT : CALL ACCEL_READ_WORD : RET

; GET_COS  (CALL &C02B) — In: A=angle  Out: HL=cos 8.8
GET_COS:
    CALL ACCEL_WAIT : PUSH AF : LD A,&21 : CALL ACCEL_CMD_START
    POP AF : OUT (C),A : CALL ACCEL_WAIT : CALL ACCEL_READ_WORD : RET

; ATAN2  (CALL &C02E) — In: HL=y DE=x  Out: A=angle(0-255)
ATAN2:
    CALL ACCEL_WAIT : LD A,&22 : CALL ACCEL_CMD_START
    CALL ACCEL_SEND_WORD : EX DE,HL : CALL ACCEL_SEND_WORD
    CALL ACCEL_WAIT : LD B,DEV : LD C,COPRO_RESULT : IN A,(C) : RET

; MAT_TRANSFORM  (CALL &C034)
; In: IX=matrix(64 bytes 16.16)  IY=vertex(16 bytes xyzw)  HL=result_buf(16 bytes)
MAT_TRANSFORM:
    CALL ACCEL_WAIT : PUSH HL : LD A,&32 : CALL ACCEL_CMD_START
    PUSH IX : POP HL : LD D,64 : CALL STREAM_N
    PUSH IY : POP HL : LD D,16 : CALL STREAM_N
    POP HL : CALL ACCEL_WAIT
    LD B,DEV : LD C,COPRO_RESULT : LD D,16
.r: IN A,(C) : LD (HL),A : INC HL : DEC D : JR NZ,.r : RET

; PERSPECTIVE  (CALL &C037) — In: IX=xyzfocal(16 bytes 16.16)  Out: HL=sx DE=sy
PERSPECTIVE:
    CALL ACCEL_WAIT : LD A,&33 : CALL ACCEL_CMD_START
    PUSH IX : POP HL : LD D,16 : CALL STREAM_N
    CALL ACCEL_WAIT
    CALL ACCEL_READ_WORD : PUSH HL
    CALL ACCEL_READ_WORD             ; hi word — discard
    CALL ACCEL_READ_WORD : EX DE,HL
    POP HL : RET
```

### Graphics routines (CALL via jump table)

```asm
; GFX_FILL_RECT  (CALL &C03A) — In: IX=10 bytes: x(2) y(2) w(2) h(2) col(2)
GFX_FILL_RECT:
    CALL ACCEL_WAIT : LD A,&01 : CALL ACCEL_CMD_START
    PUSH IX : POP HL : LD D,10 : CALL STREAM_N : RET

; GFX_BLIT  (CALL &C03D) — In: IX=12 bytes: sx sy dx dy w h (2 each)
GFX_BLIT:
    CALL ACCEL_WAIT : LD A,&02 : CALL ACCEL_CMD_START
    PUSH IX : POP HL : LD D,12 : CALL STREAM_N : RET

; GFX_LINE  (CALL &C043) — In: IX=10 bytes: x1 y1 x2 y2 col (2 each)
GFX_LINE:
    CALL ACCEL_WAIT : LD A,&30 : CALL ACCEL_CMD_START
    PUSH IX : POP HL : LD D,10 : CALL STREAM_N : RET

; GFX_TRI_Z  (CALL &C046) — non-blocking, queues to Core 1
; In: IX=30 bytes: x1 y1 z1 u1 v1 / x2 y2 z2 u2 v2 / x3 y3 z3 u3 v3 (2 each)
GFX_TRI_Z:
.q: LD B,DEV : LD C,COPRO_STATUS : IN A,(C) : BIT 7,A : JR NZ,.q
    LD A,&43 : CALL ACCEL_CMD_START
    PUSH IX : POP HL : LD D,30 : CALL STREAM_N : RET

; GFX_CLS  (CALL &C049) — In: HL=RGB555 fill colour
GFX_CLS:
    CALL ACCEL_WAIT : LD A,&45 : CALL ACCEL_CMD_START
    CALL ACCEL_SEND_WORD : RET

; SET_VGA_MODE  (CALL &C04C) — In: A=mode ID
SET_VGA_MODE:
    PUSH AF : CALL ACCEL_WAIT : LD A,&05 : CALL ACCEL_CMD_START
    POP AF : OUT (C),A : RET

; SET_TEX_ADDR  (CALL &C04F) — In: HL=addr_lo A=addr_hi (bits 16-23)
SET_TEX_ADDR:
    PUSH AF : LD B,DEV : LD C,COPRO_TEX0
    OUT (C),L : INC C : OUT (C),H : INC C : POP AF : OUT (C),A : RET

; SET_ZBUF_ADDR  (CALL &C052) — In: HL=addr_lo A=addr_hi
SET_ZBUF_ADDR:
    PUSH AF : LD B,DEV : LD C,COPRO_ZBUF0
    OUT (C),L : INC C : OUT (C),H : INC C : POP AF : OUT (C),A : RET

; WAIT_VSYNC  (CALL &C055) — wait for V9990 vertical retrace
WAIT_VSYNC:
    LD B,DEV : LD C,V9990_RSEL : LD A,V9990_RSEL : OUT (C),A
    LD C,V9990_RDATA
.v: IN A,(C) : BIT 5,A : JR Z,.v : RET

; V9990_SET_WRITE_ADDR  (CALL &C05B) — In: HL=VRAM addr (17-bit)
V9990_SET_WRITE_ADDR:
    LD B,DEV
    LD C,V9990_RSEL : LD A,32 : OUT (C),A
    LD C,V9990_RDATA : LD A,L : OUT (C),A
    LD C,V9990_RSEL : LD A,33 : OUT (C),A
    LD C,V9990_RDATA : LD A,H : OUT (C),A
    LD C,V9990_RSEL : LD A,34 : OUT (C),A
    LD C,V9990_RDATA : XOR A  : OUT (C),A : RET

; V9990_WRITE_VRAM  (CALL &C058) — In: HL=CPC src DE=VRAM dest BC=len
V9990_WRITE_VRAM:
    PUSH HL : PUSH BC : EX DE,HL : CALL V9990_SET_WRITE_ADDR
    POP BC : POP HL : LD C,V9990_VRAM
.w: LD A,(HL) : OUT (C),A : INC HL : DEC BC
    LD A,B : OR C : JR NZ,.w : RET
```

---

## 22. PSRAM Address Constants

```asm
VRAM_BASE    EQU &000000  ; V9990 VRAM  (512 KB)
FB_BASE      EQU &080000  ; VGA framebuffer  (2 MB)
ZBUF_BASE    EQU &280000  ; Z-buffer  (512 KB)
TEX_BASE     EQU &300000  ; Texture RAM  (2 MB)
BACKING_BASE EQU &500000  ; Window backing store  (3 MB)
```


---

# PART 5 — PROJECT STATUS

---

## 23. Development Roadmap

| Phase | Goal | Pass criteria |
|-------|------|--------------|
| 1 | PCB v1 designed and fabricated | All components placed and routed. Send to fab. |
| 2 | PCB bring-up — power and clock | Core2350B stable at 294 MHz. 3.3 V rail clean on scope. |
| 3 | RGB555 VGA test pattern | All 32,768 colours correct on monitor. No DAC noise. |
| 4 | Bus snooper validation | Logic analyser confirms every Z80 write cycle captured correctly. |
| 5 | RAM replace mode | CPC boots from RP2350B SRAM with RAMDIS asserted. 128 KB banking working. |
| 6 | ROM serving | 6128 BASIC boots. AMSDOS in slot 7. Accelerator RSX auto-installs from slot 5. |
| 7 | CPC native VGA output at 50 Hz | BASIC start screen reconstructed correctly on VGA. Overscan demo test. |
| 8 | V9990 port decode | OUT &FF60 writes reach PSRAM VRAM. IN &FF64 returns correct status. |
| 9 | V9990 P1 mode | SymbOS desktop loads and renders on VGA without artefacts. |
| 10 | V9990 blitter | LMMC, LMMV, LMMM working via unmodified SymbOS GFX9000 driver. |
| 11 | Coprocessor math | MUL16, DIV16, SIN, COS, MAT_TRANSFORM all return correct results. |
| 12 | 3D rasteriser | TRI_TEXTURED_Z draws correctly into VRAM, visible on VGA. |
| 13 | PCB v2 if needed | Fix any layout issues found during v1 testing. |
| 14 | Advanced display effects | Raster bar demo correct. Overscan demo fills screen. Split screen working. |
| 15 | Higher VGA modes | 800×600 and 1024×768 double-scan modes stable on monitors. |
| 16 | SymbOS full session | Extended SymbOS use stable — apps, file manager, graphics all working. |

