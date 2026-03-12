# Peopoly Magneto X - Klipper Analysis Summary

## What We Know

### Hardware Architecture
The Magneto X electronic system has 20+ PCBs across five subsystems:

| Component | Details |
|---|---|
| **Host Board** | Orange Pi Zero 2 (Allwinner H616) + Linux Hub PCB with USB-to-CAN module and 7-port USB hub |
| **Motion Controller** | BTT Octopus Pro v1.1, STM32H723, connected to host via USB Type-C |
| **Toolhead MCU** | RP2040-based CAN bus toolhead PCB (`MAG_TOOL`), UUID-addressed via `canbus_uuid` |
| **Linear Motor Driver** | ESP32-based control board, handles initialization/calibration/error for MagXY system |
| **Linear Motor Driver Module** | Converts pulse signals to 3-phase AC for coil current control |
| **Load Cell Sensor** | CS1237 ADC → STC8051 MCU → digital signal → RP2040 toolhead (GPIO24) |
| **Power** | 24V/350W (general) + 48V/600W (linear motors) |

### Klipper Config (from mypeopoly/magneto-x-klipper-config)
- **Kinematics**: Cartesian, max velocity 1500mm/s, max accel 15000mm/s²
- **MCU**: `[mcu]` via serial USB (`/dev/serial/by-id/usb-Klipper_stm32h723xx_...`)
- **Toolhead**: `[mcu MAG_TOOL]` via CAN bus (`canbus_uuid: 70e5d37dad1c`)
- **Extruder**: Step/dir on GPIO5/GPIO4, heater on GPIO0, TMC2209 UART driver
- **Load cell**: GPIO24 on toolhead, with GPIO25 reset pin, threshold-based triggering
- **Quad gantry leveling** with 4 Z steppers, 8x6 bed mesh
- **ADXL345 accelerometer** on toolhead for input shaping
- **Build volume**: 400x300x300mm

### Klipper Codebase Support (this repo)
Already present in upstream Klipper:
- **Load cell system**: `load_cell.py`, `load_cell_probe.py`, `hx71x.py`, `ads1220.py`
- **trigger_analog**: MCU-level analog trigger detection with SOS filtering (`trigger_analog.c/.py`)
- **SOS filter**: Fixed-point digital filtering for load cell noise rejection (`sos_filter.c/.h`)
- **STM32H723**: Full support including ADC, SPI, GPIO, FDCAN (`src/stm32/stm32h7*.c`)
- **CAN bus**: FDCAN driver, serial-over-CAN, UUID discovery (`fdcan.c`, `canbus.c`, `canserial.c`)
- **BTT Octopus Pro v1.1**: Board config exists (`config/generic-bigtreetech-octopus-pro-v1.1.cfg`)
- **Probe infrastructure**: Full probe system with load cell probe integration
- **Bulk sensor framework**: For high-rate sensor data streaming

### Peopoly's Klipper Fork (mypeopoly/Klipper)
- Based on Klipper V0.11 (outdated - upstream is well past this)
- Customizations focused on **loadcell communication protocol** and **linear motor driver communication**
- Only 6 commits diverging from upstream
- The loadcell approach differs from upstream: Peopoly uses CS1237 ADC + STC8051 → digital trigger, while upstream Klipper supports HX711/HX717/ADS1220 with MCU-level SOS filtering

### Known Issues (from Wiki/Community)
1. **Move queue overflow**: Reported on BTT Octopus Pro + Orange Pi Zero2 during long prints
2. **Loadcell errors**: "Endstop z still triggered after retract" and QGL internal errors
3. **Linear motor errors**: Multiple error codes (0x86, 0x11), motor status LED turning red, requires re-initialization
4. **ESP32 linear motor driver**: Requires manual firmware flashing via CH340 USB with MagnetoMotionFlashTool
5. **Pulse mode requirement**: After auto-calibration, driver must be set to Pulse Mode for Klipper control

---

## What's Left to Figure Out

### 1. Linear Motor Integration with Klipper (HIGH PRIORITY)
- **How does Klipper talk to the ESP32 linear motor driver?** The ESP32 board converts to pulse signals, but we don't know the exact interface: step/dir pulses? UART commands? Custom protocol?
- The linear motor appears to accept **step/dir pulse signals** (Pulse Mode after calibration), meaning Klipper likely treats it like a regular stepper — but this needs confirmation
- Need to examine Peopoly's Klipper fork (`mypeopoly/Klipper`) diff against upstream to see what linear motor code they added
- Is there any feedback loop from the ESP32 encoder back to Klipper?

### 2. Load Cell Signal Chain (MEDIUM PRIORITY)
- Peopoly's load cell path: **CS1237 ADC → STC8051 → digital high/low → RP2040 GPIO24**
- This is a simple digital trigger, NOT the analog load cell approach in upstream Klipper (HX711/ADS1220 → MCU analog filtering → trigger_analog)
- **Question**: Can the Magneto X work with upstream Klipper's load cell probe system, or does it need the simple digital trigger approach?
- The STC8051 firmware (`magx-loadcell-20240201.hex`) handles thresholding — DIP switches set the trigger level (default 200)
- Upstream Klipper's `trigger_analog` + `sos_filter` system is more sophisticated and could potentially replace the STC8051 thresholding if the raw ADC data were accessible

### 3. Peopoly Fork Diff Analysis (HIGH PRIORITY)
- Need to clone `mypeopoly/Klipper` and diff it against the Klipper V0.11 tag to see exactly what they changed
- This will reveal the custom communication protocols for loadcell and linear motor
- Only 6 commits — should be a small, focused diff

### 4. CAN Bus Toolhead Configuration (LOW PRIORITY)
- CAN bus support exists in upstream Klipper and the config structure looks standard
- May need minor config adjustments for the specific RP2040 toolhead pinout
- The `magneto_toolhead.cfg` has duplicate button definitions that would cause conflicts — potential bug

### 5. Firmware Flashing Workflow (LOW PRIORITY)
- BTT Octopus Pro: Standard SD card flashing (`klipper.bin` → `firmware.bin` on TF card)
- ESP32 linear motor: Requires Windows tool (MagnetoMotionFlashTool) + CH340 driver + REST button hold
- Loadcell STC8051: Separate `.hex` flashing process
- RP2040 toolhead: Likely standard Klipper CAN bootloader

### 6. Move Queue Overflow Investigation (MEDIUM PRIORITY)
- Reported by Magneto X users on BTT Octopus Pro
- Could be related to high-speed linear motor moves overwhelming the MCU
- May need tuning of move lookahead or queue size parameters

---

## Recommended Next Steps

1. **Clone and diff Peopoly's Klipper fork** against V0.11 to extract their custom code
2. **Determine linear motor step/dir interface** — confirm it's standard pulse mode
3. **Evaluate load cell approach** — decide between Peopoly's digital trigger vs upstream analog filtering
4. **Create a Magneto X board config** for upstream Klipper that maps all the pins correctly
5. **Test with upstream Klipper** on actual hardware to identify gaps
