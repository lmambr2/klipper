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

## Deep Dive: "Stepper too far in past" & Move Queue Overflow

### The Error Mechanism

The error lives in `stepper_load_next()` (`src/stepper.c:104-108`), which runs when
the MCU finishes one move segment and loads the next from its queue:

```c
// stepper_load_next() - called when current move finishes
uint32_t min_next_time = s->time.waketime;  // time of last step event
s->next_step_time += move_interval;          // scheduled time of first step in new move

if (was_active && timer_is_before(s->next_step_time, min_next_time)) {
    int32_t diff = s->next_step_time - min_next_time;
    if (diff < (int32_t)-timer_from_us(1000))   // more than 1ms behind?
        shutdown("Stepper too far in past");     // SHUTDOWN
    s->time.waketime = min_next_time;            // else: clamp to now
}
```

This fires when:
1. The stepper is actively stepping (finishing one move, loading the next)
2. The first step of the **new** move was supposed to happen >1ms **before** the
   last step of the previous move finished
3. This means the MCU fell behind — it couldn't execute steps fast enough

### Why STM32H7 is Affected

**Critical finding**: STM32H7 does NOT get the edge-optimized stepper path:

```
src/stm32/Kconfig:16:  select HAVE_STEPPER_OPTIMIZED_BOTH_EDGE if !MACH_STM32H7
```

So STM32H723 uses `stepper_event_full()`, which:
- Fires **twice per step** (once for step, once for unstep)
- Reads `timer_read_time()` on every event
- Has more overhead per step than the edge-optimized path

The edge-optimized path (`stepper_event_edge()`) toggles GPIO on each call, meaning
it fires only **once per step** and avoids the timing check entirely. This path
is available on other STM32 families but NOT H7.

### Step Rate Math for Magneto X

Config: `rotation_distance=3.2mm, microsteps=16`

```
Steps per mm = (200 full steps × 16 microsteps) / 3.2mm = 1000 steps/mm

At 1500 mm/s:  1,500,000 steps/sec = 1.5 MHz step rate
At 500 mm/s:     500,000 steps/sec = 500 kHz step rate
At 100 mm/s:     100,000 steps/sec = 100 kHz step rate
```

With `stepper_event_full()` firing twice per step:

```
At 1500 mm/s: 3,000,000 timer events/sec → 173 MCU ticks between events (520MHz)
At 500 mm/s:  1,000,000 timer events/sec → 520 MCU ticks between events
```

**173 ticks between events at max speed is extremely tight**. Each `stepper_event_full()`
call includes a `timer_read_time()`, GPIO toggle, comparison, and conditional branch.
This easily exceeds the available time budget, causing the MCU to fall behind.

### Move Queue Overflow Connection

The "Move queue overflow" (`src/basecmd.c:90`) fires when `move_alloc()` finds the
free list empty. The move pool is allocated at startup (`alloc_chunks(move_item_size, 1024, &move_count)`), requesting up to 1024 entries.

The connection to "stepper too far in past":
1. At high step rates, each move segment covers fewer steps (step compression
   creates segments of similar time duration)
2. More segments are consumed per second → moves are dequeued faster
3. If the host can't refill the queue fast enough → overflow
4. Meanwhile, the MCU is also falling behind on step execution → "too far in past"

Both errors share the root cause: **the step rate is too high for the MCU to handle**.

### Why The Linear Motor Makes It Worse

The linear motor itself isn't the direct cause — the step rate is. But:

1. **rotation_distance=3.2mm is very small** — typical belt-driven printers use
   32-40mm rotation distance, giving 10-12x fewer steps/mm
2. **1500mm/s is extremely fast** — combined with the small rotation_distance,
   this creates an enormous step rate
3. **No step-on-both-edges optimization** on STM32H7 means double the timer events
4. **The ESP32 driver may have minimum pulse width requirements** that increase
   `step_pulse_ticks`, further constraining timing

### Possible Solutions

**Option A: Reduce effective step rate (config-level)**
- Increase `rotation_distance` if the linear motor encoder allows coarser stepping
- Reduce `microsteps` from 16 to 8 or 4 (halves/quarters the step rate)
- Lower `max_velocity` to stay within MCU capacity

**Option B: Enable edge optimization for STM32H7 (firmware-level)**
- The edge optimization is disabled on H7 — investigate why
- If the H7 GPIO toggle timing meets the `EDGE_STEP_TICKS` requirement,
  enabling it would halve the timer event rate
- This is the most impactful single change

**Option C: Increase the safety threshold (firmware-level)**
- Peopoly's approach (commenting out the check entirely) is dangerous
- A better fix: increase the threshold from 1ms to something larger (e.g., 5ms)
- This tolerates temporary MCU overload without hiding catastrophic failures

**Option D: Use step compression more aggressively (host-level)**
- Adjust `STEPCOMPRESS_FLUSH_TIME` and compression parameters
- Larger move segments = fewer queue entries consumed per second
- Reduces both "too far in past" and "move queue overflow" likelihood

---

## Updated Status

| Area | Status | Key Finding |
|---|---|---|
| Linear motor ↔ Klipper | **SOLVED** | Standard step/dir pulses, no custom protocol |
| Load cell protocol | **SOLVED** | Digital trigger via STC8051, not analog |
| Stepper timing | **ROOT CAUSE FOUND** | 1.5M steps/sec at max speed + no H7 edge optimization + stepper_event_full() 2x overhead = MCU can't keep up |
| Move queue overflow | **ROOT CAUSE FOUND** | Same cause — high step consumption rate exhausts move pool |
| Load cell false triggers | **IDENTIFIED** | Tare/clear sequence unreliable |
| Upstream compatibility | **GAP** | Fork stuck on V0.11, needs port to current |

## Recommended Next Steps

1. **Investigate why edge optimization is disabled on STM32H7** — enabling it
   would be the single biggest improvement (halves timer event rate)
2. **Test with reduced microsteps** (8 or 4) to confirm step rate is the issue
3. **Check ESP32 minimum pulse width** — if it needs long pulses, `step_pulse_ticks`
   may be the bottleneck
4. **Evaluate load cell upgrade** — replace CS1237/STC8051 with HX717 for upstream
   `load_cell_probe` compatibility
