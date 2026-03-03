# AGENTS.md

## Cursor Cloud specific instructions

This is a **hardware/firmware bug-finding challenge** from comma.ai, not a traditional software project. There are no web servers, databases, or runtime services.

### Project structure
- `firmware/` — Arduino/Teensy C++ firmware (`.ino` + CY8C9560 I2C driver library)
- `kicad_files/` — KiCad 8 schematic (`.kicad_sch`) and PCB layout (`.kicad_pcb`)
- `platformio.ini` — PlatformIO build configuration (targets Teensy 4.1)

### Building firmware
```
pio run
```
Compiles the firmware for the Teensy 4.1 target. The compiled `.hex` is output to `.pio/build/teensy41/firmware.hex`. There is no physical board to upload to in this environment.

### Linting / checking
- **Firmware:** `pio check` runs static analysis on the firmware source (requires cppcheck or similar). Compilation (`pio run`) itself catches syntax and type errors.
- **Schematic ERC:** `kicad-cli sch erc /workspace/kicad_files/hardware_challenge.kicad_sch`
- **PCB DRC:** `kicad-cli pcb drc /workspace/kicad_files/hardware_challenge.kicad_pcb`

Note: ERC/DRC violations are **expected** — the project intentionally contains bugs for the challenge.

### Key caveats
- PlatformIO is installed via pip in `~/.local/bin`. Ensure `$HOME/.local/bin` is on `PATH`.
- KiCad 8 is installed from the `ppa:kicad/kicad-8.0-releases` PPA. The `kicad-cli` command is available system-wide.
- The firmware targets **Teensy 4.1** (IMXRT1062). It uses Teensy-specific features: `BUILTIN_SDCARD`, `Wire2`, `Serial1`.
- There are no automated unit tests in this repository. Validation is done through compilation and KiCad ERC/DRC.
