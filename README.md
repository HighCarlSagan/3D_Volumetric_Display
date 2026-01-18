# 3D Volumetric POV Display

A true 3D volumetric display using persistence of vision (POV) technology. Creates floating 3D images viewable from 360° without glasses or special equipment.

![Status](https://img.shields.io/badge/status-in%20development-yellow)
![Hardware](https://img.shields.io/badge/hardware-RP2040-blue)
![License](https://img.shields.io/badge/license-MIT-green)

## Overview

This project builds a cylindrical volumetric display by rapidly spinning an LED matrix (1200-1800 RPM). The human eye's persistence of vision effect merges the successive slices into a complete 3D image that appears to float in mid-air.

**Key Features:**
- True 3D visualization (not stereoscopic)
- 360° viewing angle
- Mains-powered via wireless inductive charging
- 8×12 or 16×12 LED resolution
- RP2040 microcontroller for precise timing
- Supports both matrix scanning and addressable LEDs

## Specifications

| Component | Specification |
|-----------|---------------|
| Controller | RP2040 (Raspberry Pi Pico / Waveshare RP2040-Zero) |
| LED Matrix | 8×12 (96 LEDs) or 16×12 (192 LEDs) |
| Rotation Speed | 1200-1800 RPM (20-30 FPS) |
| Power Transfer | Wireless inductive coils (5V @ 500mA) |
| Display Volume | Cylindrical (40-80mm diameter × 60mm height) |
| Position Sensing | IR photodiode + reflective target |
| Data Format | Polar coordinates (r, θ, z) |

## How It Works

1. **LED Matrix**: Vertical PCB with 8-16 columns × 12 rows of LEDs
2. **Rotation**: Entire assembly spins on motor shaft at ~30 Hz
3. **Synchronization**: IR sensor detects reference point once per rotation
4. **Display**: Shows different angular "slices" of 3D object as it rotates
5. **Persistence of Vision**: Your brain fuses the slices into solid 3D image

```
       LED Matrix (vertical)
            ●●●●●●●●  ← 8 columns
            ●●●●●●●●
            ●●●●●●●●  ← 12 rows
               ║
       ════════╩════════  Motor shaft (rotates 360°)
```

## Project Structure

```
3D_Volumetric_Display/
├── hardware/
│   ├── pcb/              # KiCad PCB designs
│   ├── mechanical/       # 3D printable parts (motor mount, enclosure)
│   └── bom.md           # Bill of materials
├── firmware/
│   ├── rp2040/          # RP2040 firmware (C/C++)
│   └── test/            # Testing utilities
├── software/
│   ├── blender/         # Scripts for generating volumetric data
│   └── tools/           # Conversion and visualization tools
├── docs/
│   ├── build_guide.html # Complete build documentation
│   └── assembly.md      # Step-by-step assembly instructions
└── README.md
```

## Quick Start

### Hardware Requirements

**Electronics:**
- Waveshare RP2040-Zero or similar (~$4)
- 96× 0805 LEDs (~$2-5)
- Wireless charging coils 5V/1A (~$10)
- IR sensor (TCRT5000) (~$1)
- Voltage regulators (AMS1117-5.0, AMS1117-3.3) (~$1)
- Small brushless motor (~$10)
- Miscellaneous (capacitors, resistors, PCB) (~$15)

**Total Cost: ~$50-80**

### Build Phases

1. **Static Test**: Verify LED matrix driving without rotation
2. **Power System**: Test wireless power transfer
3. **Mechanical**: Assemble PCBs, mount on motor shaft, balance
4. **Integration**: Add position sensing and motor control
5. **Software**: Load firmware, create volumetric content

See `docs/build_guide.html` for complete instructions.

## Design

### Matrix Scanning
- Direct GPIO control of 96 LEDs using 20 pins
- Lowest cost (~$2 for LEDs)
- Maximum timing control
- Monochrome display

## Technology Stack

- **Microcontroller**: RP2040 (dual-core ARM Cortex-M0+)
- **Language**: C/C++ (Pico SDK)
- **3D Modeling**: Blender (for content generation)
- **PCB Design**: KiCad 7+
- **Mechanical**: FreeCAD / Fusion 360 / OpenSCAD

## Safety

⚠️ **Important Safety Considerations:**
- Always use a protective enclosure (clear acrylic recommended)
- Emergency stop button accessible
- Test at low speeds before full RPM
- Wear eye protection during testing
- Ensure rotor is properly balanced

## Contributing

Contributions welcome! Areas of interest:
- Firmware optimization
- Content generation tools
- RGB color support
- Higher resolution designs
- Documentation improvements

## Resources

- [Build Guide](docs/build_guide.html) - Complete technical documentation
- [Original Inspiration](https://mitxela.com/projects/candle) - mitxela's POV Candle
- [RP2040 Datasheet](https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf)
- [Blender Volumetric Pipeline](software/blender/) - 3D content generation

## License

MIT License - See LICENSE file for details

## Acknowledgments

- Inspired by [mitxela's POV Candle](https://mitxela.com/projects/candle)
- RP2040 community for excellent documentation
- POV display pioneers and researchers

## Status & Roadmap

- [x] Initial design and documentation
- [x] PCB schematic design
- [ ] PCB layout and fabrication
- [ ] Firmware development
- [ ] Mechanical assembly
- [ ] Testing and calibration
- [ ] Content generation pipeline
- [ ] Full system integration

---

**Author**: Mayank S
**Project Start**: January 2026  
**Status**: Active Development
