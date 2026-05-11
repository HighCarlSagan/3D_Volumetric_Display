# VOL-1 — Volumetric POV Display

**A vertical-blade volumetric display in the spirit of mitxela's Candle. 12 × 12 APA102C matrix on a spinning PCB, inductively powered from a stationary base, dual-IR phase lock.**

[![Status](https://img.shields.io/badge/status-design_doc-blue)](docs/build-roadmap.html)
[![License](https://img.shields.io/badge/license-CERN--OHL--P--2.0-blue)](LICENSE)
[![Firmware](https://img.shields.io/badge/firmware-ESP--IDF_%2B_STM32_HAL-red)](#firmware)
[![Designed With](https://img.shields.io/badge/Open_Hardware-OSHW-orange)](https://www.oshwa.org/)

<p align="center">
  <em>TODO: hero shot — long-exposure photo of the spinning blade once hardware exists</em><br/>
  <code>media/hero.jpg</code>
</p>

---

> **Project status: design doc, no hardware yet.**
> This README describes the *intended* design. No PCB has been fabricated. No firmware has been written. Everything below is the plan — see [`docs/build-roadmap.html`](docs/build-roadmap.html) for the detailed build roadmap (rev B).

---

## Why I'm Building This

I have wanted to build a persistence-of-vision volumetric display for years. The reference point in my head has always been [mitxela's Candle](https://mitxela.com/projects/candle) — a hand-sized object that quietly does something the eye doesn't quite know how to process, and looks beautiful while doing it. POV displays sit at the intersection of mechanical engineering, power electronics, real-time embedded code, and a little bit of optics — which is to say, everything I like about hardware in one project.

VOL-1 is also a deliberate scope test. Versus OakBridge (PCB + firmware + companion app, all stationary, all wired), VOL-1 adds **a motor**, **a wireless power link**, **two independent optical sync paths**, and **two MCUs that share nothing but light**. If I can make that work cleanly end-to-end, the bar for everything else I want to build goes up.

The previous attempt at this project — under the same repo, before it had a codename — was an RP2040 driving a matrix-scanned grid of 0805 LEDs on a single board. That design is dead. VOL-1 starts from a different architecture, the right one this time.

---

## Concept

A thin vertical PCB — the **rotor blade** — carries a 12 × 12 APA102C matrix on its front face. The blade mounts onto a motor shaft through a hole at its centerline. The motor sits in a stationary cylindrical **base** below. When the motor spins, the blade rotates about its own vertical axis, and each row of LEDs sweeps a circle at its height. 144 stacked circles produce a hollow cylindrical voxel volume that the eye perceives as a solid 3D image.

Two physical domains:
- **The stationary base** handles AC power, motor drive, motor speed control, and an IR emitter for rotor sync.
- **The spinning rotor** handles LEDs and rendering.

They communicate only through:
- A **Qi-style inductive power link** (base → rotor)
- **Two independent IR optical paths** (one each direction) for angular alignment

No slip rings. No wires across the rotating boundary.

---

## Initial Requirements

| # | Requirement | Target | Status |
|---|---|---|---|
| 1 | Voxel volume | ≥10k addressable voxels | 🎯 ~17.3k planned (144 LEDs × 120 angular slices) |
| 2 | Rotor speed | Stable enough for flicker-free perception | 🎯 1800 RPM = 30 Hz, PI-locked |
| 3 | Power transfer | Wireless, no slip rings | 🎯 Qi-compatible 5 W inductive link, 1–3 mm gap |
| 4 | Angular sync | Independent, electrically isolated | 🎯 Dual IR optical (one each direction) |
| 5 | LED tech | Addressable RGB, fast SPI | 🎯 APA102C-2020 (no PWM jitter, ≥20 MHz SPI) |
| 6 | Rotor MCU | WiFi + dual-core + SPI DMA | 🎯 ESP32-S3-MINI-1 (N8R2) |
| 7 | Base MCU | Cheap, interrupt-driven, ≥32-bit | 🎯 STM32G031K8 |
| 8 | Safety | Full blade containment at speed | 🎯 Clear acrylic dome, 70 mm ID × 130 mm |
| 9 | Power input | Single USB-C | 🎯 USB-C 2.0, TPS54302 buck to 5 V |
| 10 | Content pipeline | Live shape switching over network | 🎯 WiFi POST endpoint → C++ Shape classes |
| 11 | Documentation | Build-reproducible from this repo | 🎯 Gerbers + STLs + firmware + BOM published |

Nothing is built. Outcomes will be filled in as bring-up happens.

---

## Specifications (target)

| | |
|---|---|
| **Codename** | VOL-1 (MkI) |
| **Topology** | Vertical-blade rotating POV |
| **Voxel count** | ~17,280 (144 LEDs × 120 angular slices) |
| **Rotor speed** | 1800 RPM (30 Hz) |
| **LED matrix** | 12 × 12 APA102C-2020 RGB |
| **Blade dimensions** | 15 × 100 × 1.6 mm |
| **Swept volume** | Ø 15 × 100 mm cylinder |
| **Rotor MCU** | ESP32-S3-MINI-1 (N8R2) — 8 MB flash, 2 MB PSRAM |
| **Base MCU** | STM32G031K8T6 — 64 MHz Cortex-M0+ |
| **Power link** | Qi-compatible 5 W inductive, 125 kHz, 1–3 mm gap |
| **Sync** | Two independent IR paths (dual-MCU, optically coupled) |
| **Input power** | USB-C 5 V, 3 A via TPS54302 buck |
| **Rotor power budget** | ~4 W average, 7 W peak (brightness-capped) |
| **Enclosure** | 3D-printed PETG base, clear acrylic dome |
| **Estimated BOM cost** | ~₹6,600 (one-off prototype, India) |

Full bandwidth, power, and balance derivations in [`docs/build-roadmap.html`](docs/build-roadmap.html).

---

## Architecture

```
ROTOR BLADE  (spins at 1800 RPM)
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│         12 × 12 APA102C matrix  (front face)                │
│                  │                                          │
│                  │ SPI @ 20 MHz                             │
│                  ▼                                          │
│         ESP32-S3-MINI-1  ◄── IR sync pulse (phototransistor │
│         (dual-core, WiFi)        pointed DOWN, through hole │
│                  ▲               in PCB underside)          │
│                  │ 3.3 V                                    │
│         RT9013 LDO                                          │
│                  ▲                                          │
│                  │ 5 V                                      │
│         Bridge rectifier + 1000 µF bulk                     │
│                  ▲                                          │
│         RX coil (horizontal, near shaft)                    │
└─────────────────│───────────────────────────────────────────┘
                  │
                  │  ≈ ≈ ≈  Qi-compatible 5 W inductive link
                  │         125 kHz, 1–3 mm gap
                  │
┌─────────────────│───────────────────────────────────────────┐
│         TX coil (flush top face of base)                    │
│                  ▲                                          │
│         Class-E driver (Qi TX module or discrete)           │
│                  ▲                                          │
│         5 V rail ┼──► Brushed DC motor ◄── N-ch MOSFET      │
│                  │    (RF-310)               ▲              │
│                  │                           │ PWM          │
│                  │                  STM32G031K8             │
│                  │                           ▲              │
│                  │                           │              │
│                  │                  TCRT5000 IR sensor      │
│                  │                  (aimed UP at rotor      │
│                  │                  underside reflector tab)│
│                  │                                          │
│                  │                  IR LED (aimed UP        │
│                  │                  through aperture)       │
│                                                             │
│         USB-C ──► TPS54302 buck ──► 5 V rail                │
└─────────────────────────────────────────────────────────────┘
BASE PCB  (stationary)
```

### The two independent IR paths

This is the part most people get wrong. The base has two IR components doing two different jobs, and the rotor has one doing a third:

| Where | Component | Direction | Purpose |
|---|---|---|---|
| **Base** | IR LED (TSAL6200) | points **UP** | Continuous-on optical beacon for the rotor |
| **Base** | TCRT5000 reflective sensor | points **UP** | Reads a white-soldermask reflector tab on rotor underside — **motor speed feedback** |
| **Rotor** | Phototransistor (TEFT4300) | points **DOWN** | Sees base's IR LED once per rev through cutout — **image angular sync** |

The two paths are completely decoupled electrically. They happen to fire near-simultaneously each revolution because the reflector tab and the IR window are at the same angular position on the rotor — but nothing in firmware depends on that coincidence. Each MCU uses its own IR pulse independently.

---

## Why X over Y

A handful of design decisions on this build had real alternatives. Capturing them here so future-me remembers the trade.

**1800 RPM, not 6000 RPM.** Reference designs and the "more is more" instinct push toward 6000 RPM for higher refresh rate. I chose 1800 because: (1) it's **11× easier on balance and bearings** (centripetal force scales with ω²), (2) mitxela's Candle runs at ~1800 RPM and looks great, (3) 30 Hz × 120 angular slices is well above the perceptual flicker threshold, (4) the safety implications of a 1800 RPM blade are dramatically lower than 6000. 3000 RPM remains a post-MkI upgrade if the mechanical build turns out clean.

**APA102C over WS2812B / SK9822.** APA102C uses a real SPI clock — no protocol jitter, no microsecond-accurate timing required, and DMA-friendly. WS2812B is timing-critical and would eat ISR budget on a rotating MCU that already has hard real-time constraints. SK9822 is APA102C-compatible but rarer. APA102C in 2020 package is the right answer.

**ESP32-S3 + STM32G031 (two MCUs), not one MCU with slip rings.** Slip rings at 1800 RPM wear, introduce electrical noise, and add a single point of mechanical failure. Modulating data onto the inductive link is elegant but adds firmware complexity on both ends — not worth it for a once-per-rev signal. Two cheap optoelectronic paths is the right engineering trade.

**Qi-compatible coils, not a custom inductive design.** I could wind my own coils and design a custom Class-E driver. I'd rather spend that time on the rotor balance problem (which is unique to this project) than re-derive a power-transfer topology that already has a $3 reference solution. Würth or Adafruit Qi RX/TX coil pair, off-the-shelf or simple TX module.

**Two separate firmwares, not a shared codebase.** The rotor and base MCUs share nothing — no code, no build system, no test harness. They communicate only through light. A monorepo with a shared `common/` directory is exactly the kind of false convenience that creates accidental coupling. Each MCU has its own folder, its own `main.c` / `rotor.cpp`, and its own definition of "the protocol" (in this case: "an IR pulse arrived").

**Shape-as-a-function, not voxel-file-format.** Each renderable shape is a C++ class implementing `RGB sample(float theta, float y)`. No voxel buffers to upload, no GLSL, no asset pipeline. Add a new shape by writing a class and flashing the firmware. WiFi endpoint switches between compiled-in shapes at runtime. Trades flexibility (no user-uploadable content) for simplicity (no file format, no parser, no validation) — the right call for MkI.

**Brightness capped at 25% globally in firmware.** 144 LEDs × 20 mA at full white = 2.88 A peak. No 5 W inductive link will deliver that. The cap is in firmware, enforced unconditionally, and protects the rotor rail from collapsing during bright frames.

---

## Key Components

### Rotor

| Component | Part | Function |
|---|---|---|
| MCU | ESP32-S3-MINI-1 (N8R2) | Dual-core, WiFi, 8 MB flash, 2 MB PSRAM |
| LEDs | APA102C-2020 (×144) | 2 × 2 mm RGB addressable, SPI-clocked |
| RX coil | Würth 760308101312 | Qi 5 W receiver, 10 µH |
| Rectifier | MB6S | 0.5 A Schottky bridge |
| 3.3 V LDO | RT9013-33GB | 500 mA, low dropout |
| Phototransistor | TEFT4300 | IR sync receiver, 950 nm |
| Bulk cap | 1000 µF / 10 V low-ESR | Rectifier smoothing |

### Base

| Component | Part | Function |
|---|---|---|
| MCU | STM32G031K8T6 | 64 MHz M0+, motor PI loop |
| Motor | RF-310FA-11400 | Brushed DC, 3–6 V |
| Motor MOSFET | AO3400A | Logic-level N-ch, 5.8 A |
| TX coil | Würth 760308100110 | Qi 5 W transmitter, 6.3 µH |
| TX driver | Adafruit #1459 or discrete Class-E | 125 kHz Qi-compatible |
| TCRT5000 | Vishay TCRT5000L | Reflective speed sensor |
| Base IR LED | TSAL6200 | 940 nm, ~100 mA, optical beacon |
| Buck | TPS54302DDCR | USB → 5 V, 3 A |
| USB-C | GCT USB4105-GF-A | UFP with CC pull-downs |

Full BOM with LCSC part numbers and JLC-ready ordering details in [`docs/build-roadmap.html`](docs/build-roadmap.html).

---

## Firmware

Two independent codebases. They share no code.

### Base — STM32G031

**Language:** C, bare-metal STM32Cube HAL or libopencm3. No RTOS. Interrupt-driven. Target binary <16 KB.

The PI motor controller runs **inside the IR EXTI ISR**. The natural control rate equals the mechanical sample rate (one update per revolution = 30 Hz at 1800 RPM), which is fast enough for a motor with ~100 ms mechanical time constant. No separate PID task needed.

```c
// Pseudocode — full skeleton in docs/build-roadmap.html
void EXTI0_1_IRQHandler(void) {
  uint32_t now = TIM2->CNT;
  period_ticks = now - last_ir_tick;
  last_ir_tick = now;
  rpm = 60.0f * TIM2_HZ / (float)period_ticks;

  float err = RPM_SETPOINT - rpm;
  integral_err = clamp(integral_err + err, -10000, +10000);
  pwm_duty = clamp(pwm_duty + KP*err + KI*integral_err, 0.10, 0.95);
  TIM1->CCR1 = pwm_duty * TIM1->ARR;
}
```

### Rotor — ESP32-S3

**Language:** C++ on ESP-IDF (Arduino framework acceptable for MkI). Dual-core, FreeRTOS.

- **Core 1** — timing-critical ISRs (IR sync, column advance, SPI DMA push)
- **Core 0** — render task and WiFi server

The framebuffer is `[angular_slice][row][rgba]` — ~2.8 KB, fits in DRAM. Each angular slice is one column of 12 LEDs that gets DMA-pushed to the APA102 chain when the column-advance timer fires. The IR sync ISR resets the column counter and reloads the column timer based on measured period.

```cpp
// Pseudocode — full skeleton in docs/build-roadmap.html
void IRAM_ATTR ir_sync_isr() {
  uint32_t now = esp_timer_get_time();
  t_period_us = now - t_last_sync_us;
  t_last_sync_us = now;
  current_col = 0;
  column_timer_set_interval(t_period_us / COLS_ANGULAR);
}

void IRAM_ATTR column_tick_isr() {
  spi_dma_burst(&framebuffer[current_col][0][0], ROWS * 4);
  current_col = (current_col + 1) % COLS_ANGULAR;
}
```

### Shape system

Each renderable shape implements:

```cpp
class Shape {
public:
  virtual RGB sample(float theta, float y) = 0;
};
```

Spheres, helices, text rendering, Lissajous figures, waveform plots — all the same interface. The render task walks the framebuffer once per ~33 ms and calls `sample()` for each (θ, y) coordinate. Active shape is a pointer that the WiFi endpoint can swap atomically at runtime.

---

## Mechanical

The mechanical stack must satisfy four simultaneous constraints:

1. **Rigid motor mount** holding the shaft perpendicular to the base within ±0.2°
2. **Coil-to-coil gap of 1–3 mm** held constant across all rotation angles
3. **Rotor balance** to within a gram-mm at the worst radius
4. **Safety enclosure** that fully contains the blade if a solder joint fails at speed

Critical mechanical part is the **3D-printed rotor hub** connecting motor shaft to PCB. Collet-style clamp, M2 set screw onto a filed flat on the motor shaft, threadlocker, PETG at 100% infill. This is where most amateur POV builds fail and where most of the design time goes.

Full mechanical dimensioning, ASCII stack-up, balance analysis, and the "every gram matters" trim-weight workflow in [`docs/build-roadmap.html`](docs/build-roadmap.html).

> ⚠️ **Safety:** Do not run the rotor outside the acrylic dome above 300 RPM. A solder joint failure at 1800 RPM launches a 2020-size LED at ~14 m/s radially. The dome is eye protection.

---

## Roadmap

The detailed day-by-day build plan lives in [`docs/build-roadmap.html`](docs/build-roadmap.html) (rev B). High-level phases:

### Phase 1 — Schematic & order (Week 1)
- Rotor schematic + layout (15 × 100 mm blade)
- Base schematic + layout (Ø 78 mm disk)
- Mechanical CAD: enclosure, top lid, motor pocket, rotor hub
- Order PCBs (JLC) + parts (LCSC + Robu.in)

### Phase 2 — Build & bring-up (Week 2)
- 3D print mechanical parts during PCB lead time
- Rotor bring-up static (bench supply, bypass inductive)
- Base bring-up static (PWM verify, TCRT5000 verify, inductive link bench test)
- Mechanical assembly, static balance with trim solder

### Phase 3 — Integrate & ship (Week 3)
- RPM ramp under inductive power
- PID lock at 1800 RPM
- IR sync verification (single line should hang motionless in space)
- Shape implementations: sphere, helix, text, Lissajous, RGB checker
- WiFi `/shape` endpoint
- Long-exposure + slow-mo demo footage
- README and gerbers published

### Risks

The roadmap calls out seven risks ranked CRIT / HIGH / MED / LOW. The headline three:

- **CRIT — balance is the #1 project killer.** Iterative static balancing with trim solder is non-negotiable. Test every intermediate speed.
- **HIGH — inductive power is tight.** Qi 5 W coils deliver ~3.5 W at 2 mm gap in practice. Firmware brightness cap protects the rotor rail.
- **HIGH — 3-week timeline assumes PCB lands on D10–11.** JLC to Bangalore can slip. Bench-wired rotor prototype as fallback.

### Definition of done

- Stable 1800 ±20 RPM for 30 minutes continuous, no bearing whine
- All 144 APA102C LEDs driven, brightness-capped
- IR sync locked: stationary vertical line hangs motionless in space
- ≥4 shapes implemented, switchable over WiFi
- Inductive rail ≥4.7 V under full display load
- Dome installed during all spin tests above 300 RPM
- Public README with hero photo, demo video, BOM, gerbers, STLs, firmware binaries, build guide

### MkII / future

Not committed. Possibilities once MkI is shipping:

- 3000 RPM operation (mechanical balance permitting)
- Larger blade (16 × 16 matrix)
- Multi-blade configurations
- Voxel file format + offline rendering pipeline
- Custom Class-E driver (replacing the Qi TX module)

---

## Why MkI starts here (and not where it was)

The repo previously contained a different design: an **RP2040 driving an 8 × 12 or 16 × 12 matrix-scanned grid of 0805 LEDs**, with a TCRT5000 sensing position and inductive power transfer. That design died for three reasons:

1. **Matrix-scanned LEDs flicker visibly** when the persistence window is short. Persistence time at 1800 RPM with N rows scanned sequentially leaves each row lit for ~1/N of the column dwell — fine for a stationary matrix, bad for a rotating one.
2. **The RP2040 has no native wireless.** Adding WiFi for live shape switching meant a second module and the kind of bolt-on complexity I wanted to avoid.
3. **0805 LEDs are too large** for a 12 × 12 grid on a 15 mm-wide blade. Migrating to APA102C-2020 was the natural follow-on.

VOL-1 is the redesign that starts from those lessons. Same problem statement, different architecture. The old design files are still in git history if anyone wants archaeology.

---

## Repository Structure

```
3D_Volumetric_Display/
├── README.md
├── LICENSE
├── hardware/
│   ├── rotor/                 # KiCad 8 project
│   │   ├── vol1_rotor.kicad_pro
│   │   ├── gerbers/
│   │   └── bom.csv            # LCSC part numbers, JLC-ready
│   └── base/                  # KiCad 8 project
├── mechanical/
│   ├── STL/                   # 3D-printable parts
│   │   ├── base_enclosure.stl
│   │   ├── base_top_lid.stl
│   │   ├── rotor_hub.stl
│   │   └── dome_retainer_ring.stl
│   └── source/                # Fusion / OnShape / FreeCAD originals
├── firmware/
│   ├── base/                  # STM32CubeIDE project, STM32G031
│   │   ├── Core/Src/main.c
│   │   ├── Core/Src/pid.c
│   │   └── Makefile
│   └── rotor/                 # ESP-IDF project, ESP32-S3
│       ├── main/
│       │   ├── rotor.cpp
│       │   ├── shapes.cpp
│       │   ├── apa102_spi.c
│       │   └── wifi_server.cpp
│       ├── CMakeLists.txt
│       └── sdkconfig.defaults
├── docs/
│   ├── build-roadmap.html     # Full build plan, rev B (this is the deep reference)
│   ├── design-notes.md        # Derivation of timing, power budget
│   ├── bringup-log.md         # Chronological debug journal (post-bringup)
│   └── photos/
└── media/
    ├── demo.mp4               # Normal + long-exposure + slow-mo
    └── hero.jpg
```

Most folders are empty for now. They will populate as the build progresses.

---

## When to Use This / When NOT to Use This

**Use this if you want to:**
- Build a small, mechanical, beautiful thing that does volumetric POV from scratch
- Learn dual-MCU coordination over an optical link
- Learn Qi-style inductive power on a rotating system
- Have a starting reference for "minimum viable POV display" without paying $500+ for a commercial unit

**Don't use this if you want to:**
- A safe, finished, ready-to-buy product. This is a personal build. The blade is a hazard outside the dome.
- High frame rates. 30 Hz is plenty for the eye. If you need 60+ Hz volumetric content, you want a different topology.
- A general-purpose volumetric image viewer. There's no file format and no upload pipeline. Shapes are compiled-in C++ classes.

---

## License

**Hardware:** [CERN Open Hardware Licence Version 2 — Permissive (CERN-OHL-P-2.0)](LICENSE)
**Firmware** (when it exists): MIT
**Documentation:** CC-BY-4.0

Free to use, modify, fabricate, and distribute with attribution. No reciprocity required.

---

## Acknowledgments

- **[mitxela](https://mitxela.com/projects/candle)** — the Candle is the spiritual reference design for this whole project. Go look at it.
- The Qi Wireless Power Consortium — for making inductive power transfer a commodity.
- Everyone who has built a POV display and written about it on Hackaday, particularly the ones that document the failure modes.

---

## Related Projects

- **[OakBridge_MkI](https://github.com/HighCarlSagan/OakBridge_MkI)** — sibling hardware project, stationary cousin. Where OakBridge is a tactile dashboard, VOL-1 is a sculptural display. Both share the "small, beautiful, electrically interesting" design brief.
- **[Carls_Homelab_Public](https://github.com/HighCarlSagan/Carls_Homelab_Public)** — where the WiFi shape-switching endpoint will eventually point.

---

## Author

**Mak (Mayank Shrivastava)** — [GitHub @HighCarlSagan](https://github.com/HighCarlSagan) · [highcarlsagan.dev](https://highcarlsagan.dev)
