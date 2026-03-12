# Peopoly Magneto X - Klipper Analysis Summary

## What We Know

### Hardware Architecture
The Magneto X has 20+ PCBs across five subsystems:

| Component | Details |
|---|---|
| **Host Board** | Orange Pi Zero 2 (Allwinner H616) + Linux Hub PCB with USB-to-CAN module and 7-port USB hub |
| **Motion Controller** | BTT Octopus Pro v1.1, STM32H723, connected to host via USB Type-C |
| **Toolhead MCU** | RP2040-based CAN bus toolhead PCB (`MAG_TOOL`), UUID-addressed via `canbus_uuid` |
| **Linear Motor Driver** | ESP32-based control board for MagXY magnetic levitation system |
| **Linear Motor Driver Module** | Converts pulse signals to 3-phase AC for coil current control |
| **Load Cell Sensor** | CS1237 ADC → STC8051 MCU → digital signal → RP2040 toolhead (GPIO24) |
| **Power** | 24V/350W (general) + 48V/600W (linear motors) |

### Klipper Config (from mypeopoly/magneto-x-klipper-config)
- **Kinematics**: Cartesian, max velocity 1500mm/s, max accel 15000mm/s²
- **MCU**: `[mcu]` via serial USB (`/dev/serial/by-id/usb-Klipper_stm32h723xx_...`)
- **Toolhead**: `[mcu MAG_TOOL]` via CAN bus (`canbus_uuid: 70e5d37dad1c`)
- **Extruder**: Step/dir on GPIO5/GPIO4, heater on GPIO0, TMC2209 UART driver
- **Load cell**: GPIO24 on toolhead, with GPIO25 reset pin, threshold-based digital triggering
- **Quad gantry leveling** with 4 Z steppers, 8x6 bed mesh
- **ADXL345 accelerometer** on toolhead for input shaping
- **Build volume**: 400x300x300mm

### Upstream Klipper Codebase Support (this repo)
Already present:
- **Load cell system**: `load_cell.py`, `load_cell_probe.py`, `hx71x.py`, `ads1220.py`
- **trigger_analog**: MCU-level analog trigger detection with SOS filtering (`trigger_analog.c/.py`)
- **SOS filter**: Fixed-point digital filtering for load cell noise rejection (`sos_filter.c/.h`)
- **STM32H723**: Full support including ADC, SPI, GPIO, FDCAN (`src/stm32/stm32h7*.c`)
- **CAN bus**: FDCAN driver, serial-over-CAN, UUID discovery (`fdcan.c`, `canbus.c`, `canserial.c`)
- **BTT Octopus Pro v1.1**: Board config exists (`config/generic-bigtreetech-octopus-pro-v1.1.cfg`)
- **Probe infrastructure**: Full probe system with load cell probe integration
- **Bulk sensor framework**: For high-rate sensor data streaming

---

## Peopoly's Klipper Fork - Full Diff Analysis

Cloned from `mypeopoly/Klipper`. The fork is based on an upstream Klipper snapshot
close to V0.11, squashed into a single init commit. All Peopoly-specific changes
are spread across **4 files** (plus 1 community plugin):

### 1. `klippy/extras/magneto_load_cell.py` (NEW FILE - 70 lines)

The **only truly new file**. A simple digital pin controller for load cell reset:

```
class PrinterLoadCellDigitalOut:
    - Configures a digital_out pin (GPIO25 on RP2040 toolhead)
    - Registers 3 GCode commands: LC28 (clear), LL28 (set low), LH28 (set high)
    - clear_load_cell(): pulls pin LOW for ~400ms, then HIGH again
    - Used as a "tare" / reset signal to the STC8051 loadcell MCU
```

**Key insight**: The load cell is NOT an analog sensor in Peopoly's design. The
STC8051 microcontroller handles all ADC reading and thresholding. It outputs a
simple digital high/low to the RP2040's GPIO24. The reset pin (GPIO25) is used
to tare/clear the load cell by toggling it low then high.

### 2. `klippy/extras/homing.py` (MODIFIED - 3 lines)

```diff
- raise self.printer.command_error("Probe triggered prior to movement")
+ #raise self.printer.command_error("Probe triggered prior to movement")
+ self.gcode.respond_info("Probe triggled prior to movement!!")
```

**Purpose**: Converts a fatal "probe already triggered" error into a warning
message. This is a **workaround** for the loadcell sometimes being in a triggered
state when probing begins — a known Magneto X issue.

### 3. `klippy/extras/probe.py` (MODIFIED - ~15 lines)

Changes:
- Adds `import time` (unused leftover)
- Adds `magneto_load_cell` lookup in ProbeEndstopWrapper (commented out —
  the load cell clear was attempted but abandoned)
- Adds `name` field to `get_status()` return dict
- Makes `horizontal_move_z` overridable per-command via `HORIZONTAL_MOVE_Z`
  GCode parameter (useful for QGL/bed mesh with different Z heights)

### 4. `src/stepper.c` (MODIFIED - 2 lines)

```diff
- if (diff < (int32_t)-timer_from_us(1000))
-     shutdown("Stepper too far in past");
+ // if (diff < (int32_t)-timer_from_us(1000))
+ //     shutdown("Stepper too far in past");
```

**Purpose**: Disables the safety shutdown when a stepper step event is more than
1ms behind schedule. This is a **critical workaround** — the linear motors likely
cause timing issues that trigger this shutdown. The move queue overflow errors
reported by users are related: the MCU can't keep up with the commanded step rate.

### 5. `klippy/extras/gcode_shell_command.py` (NEW FILE - community plugin)

Standard community plugin by Eric Callahan (not Peopoly-specific). Allows running
shell commands from GCode.

---

## What This Tells Us

### The Linear Motor Integration is Simpler Than Expected
Peopoly treats the linear motors as **standard steppers**. There is no custom
protocol, no UART/SPI communication with the ESP32, and no encoder feedback loop
in Klipper. The ESP32 driver board:
1. Self-initializes and auto-calibrates on power-up
2. Is switched to "Pulse Mode" (step/dir)
3. From that point, Klipper drives it exactly like a TMC stepper — step/dir pulses

The only evidence of issues is the **commented-out "Stepper too far in past"
shutdown**, which suggests the linear motor driver introduces timing jitter or
latency that a normal stepper doesn't have.

### The Load Cell is a Digital Trigger, Not Analog
The entire load cell "intelligence" lives in the STC8051 firmware:
- CS1237 ADC reads the strain gauge
- STC8051 applies threshold (set via DIP switches, default 200)
- Outputs HIGH/LOW to RP2040 GPIO24
- Klipper sees it as a simple endstop switch

This is fundamentally different from upstream Klipper's load cell approach, which
reads raw ADC values (via HX711/ADS1220) and applies sophisticated SOS filtering
on the MCU for better noise rejection and tap detection.

### Peopoly's Workarounds Reveal the Real Issues
1. **"Stepper too far in past" disabled** → Linear motor timing/latency problem
2. **"Probe triggered prior to movement" downgraded** → Load cell false triggers
3. **Load cell clear attempted in probe.py** → Tare issues between probing moves
4. **Move queue overflow** (user reports) → Step rate exceeds MCU capacity at high speeds

---

## What's Left To Do

### HIGH PRIORITY

1. **Investigate "Stepper too far in past" root cause**
   - Is the linear motor driver adding latency to step pulses?
   - Can this be fixed by adjusting step timing parameters rather than disabling the safety check?
   - Check if upstream's move lookahead or step compression helps

2. **Move queue overflow at high speeds**
   - At 1500mm/s with high acceleration, the step rate may overwhelm the STM32H723
   - Investigate step compression, step-on-both-edges, and queue size tuning
   - May need to profile the MCU load during high-speed linear motor moves

3. **Load cell false trigger investigation**
   - Why does the load cell report "triggered" before probing begins?
   - Is this a timing issue with the STC8051 clear/tare sequence?
   - Could upstream's analog load cell approach (HX711 + SOS filter) replace the digital trigger entirely?

### MEDIUM PRIORITY

4. **Evaluate replacing CS1237/STC8051 with HX711/HX717**
   - Would give Klipper direct access to analog force data
   - Enables upstream's sophisticated tap detection algorithm
   - Eliminates the STC8051 as a failure point
   - Hardware modification required — is it worth it?

5. **Create proper Magneto X board/printer config for upstream Klipper**
   - Map all pin assignments from Peopoly's config
   - Use upstream's probe infrastructure instead of workarounds
   - Test with latest Klipper (not stuck on V0.11)

### LOW PRIORITY

6. **Horizontal_move_z per-command override**
   - Peopoly's probe.py patch to allow GCode override of `horizontal_move_z` is
     actually useful — could be submitted as an upstream PR

7. **ESP32 linear motor driver documentation**
   - Document the initialization/calibration sequence
   - Document error codes and recovery procedures
   - Understand what happens when the ESP32 loses step sync

---

## Summary Table

| Area | Status | Key Finding |
|---|---|---|
| Linear motor ↔ Klipper | **SOLVED** | Standard step/dir pulses, no custom protocol |
| Load cell protocol | **SOLVED** | Digital trigger via STC8051, not analog |
| Stepper timing issue | **IDENTIFIED** | Safety check disabled — needs real fix |
| Move queue overflow | **IDENTIFIED** | High step rates at 1500mm/s overwhelm MCU |
| Load cell false triggers | **IDENTIFIED** | Tare/clear sequence unreliable |
| Upstream compatibility | **GAP** | Fork stuck on V0.11, needs port to current |
