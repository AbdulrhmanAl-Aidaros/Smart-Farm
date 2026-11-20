# 🌱 Smart Farm — PIC16F877A Smart Irrigation System

An embedded system that waters a farm **intelligently**: it irrigates three zones one after another, only when the soil is dry and water is available, and shuts everything down with an alarm if something goes wrong. Built on a **PIC16F877A** microcontroller, programmed in **C (MPLAB XC8)** and verified in **Proteus** simulation.

![MCU](https://img.shields.io/badge/MCU-PIC16F877A-red)
![Compiler](https://img.shields.io/badge/compiler-MPLAB%20XC8%203.10-blue)
![Simulation](https://img.shields.io/badge/simulation-Proteus-green)
![Language](https://img.shields.io/badge/language-C-lightgrey)

---

## 📌 Overview (the simple version)

Imagine a small control box with a keypad, a screen, and a few indicator lights:

- Press a key to choose how to water: **Manual**, **Auto**, or **Scheduled**.
- The farm has **3 irrigation zones**. The system waters Zone 1, then Zone 2, then Zone 3, and can repeat the whole round several times.
- **Auto mode** checks two sensors first: *is the soil dry?* and *is there water in the tank?* — it only starts if both answers are yes.
- If the **emergency switch** is pressed, or **water runs out** during irrigation, everything stops, an **alarm** turns on, and the system stays locked until you reset it.
- A **16×2 LCD** shows what is happening, and a **2-digit 7-segment display** shows the countdown (or an error code during an alarm).

> Sensors and the pump/valves are represented by **switches and LEDs** in the simulation (pump LED, 3 zone LEDs, alarm LED, status LED).

---

## ✨ Features

| Feature | Description |
|---|---|
| 🖐️ **Manual mode** | Turn Zone 1/2/3 on or off directly from the keypad |
| 🤖 **Auto mode** | Starts irrigation only if soil is dry **and** water is available |
| ⏱️ **Schedule mode** | Waits a configurable delay, then irrigates if conditions are met |
| ⚙️ **Settings mode** | Edit zone durations (T1, T2, T3), number of cycles, and the start delay |
| 🚨 **Safety layer** | Emergency switch + low-water detection with a **latched alarm** |
| 🔢 **Error codes** | `01` = Emergency, `02` = Low water (on the 7-segment display) |
| 📟 **User interface** | 16×2 LCD (4-bit mode) + 4×4 keypad |
| ⏲️ **Accurate timing** | Timer1 interrupt (1 s time base, display multiplexing, blink) |

---

## 🧰 Hardware

| Unit | Component |
|---|---|
| Processing | PIC16F877A, 8 MHz crystal (HS oscillator) |
| Input | 4×4 keypad; 3 switches simulating soil-dry sensor, water-level sensor, emergency switch |
| Display | 16×2 character LCD; 2-digit common-cathode 7-segment display |
| Output | LEDs: pump, zone 1, zone 2, zone 3, alarm, status |

![Proteus schematic — PIC16F877A](images/proteus_pic_schematic.png)

![Output LEDs](images/proteus_led_outputs.png)

### Pin mapping

| Port | Usage |
|---|---|
| **RA0** | Soil-dry sensor (input) |
| **RA1** | Water-available sensor (input) |
| **RA2** | Emergency switch (input) |
| **RA3, RA5, RE0–RE2, RC6, RC7** | 7-segment segments A–G |
| **RD2, RD3** | 7-segment digit select (multiplexing, active-low) |
| **RB0–RB3 / RB4–RB7** | Keypad columns (outputs) / rows (inputs, internal pull-ups) |
| **RD0, RD1, RD4–RD7** | LCD `RS`, `EN`, `D4–D7` (4-bit mode) |
| **RC0** | Pump LED |
| **RC1, RC2, RC3** | Zone 1, 2, 3 LEDs |
| **RC4** | Alarm LED |
| **RC5** | Status LED (blinks while running) |

---

## 🎮 How to use

### Main menu
![LCD home menu](images/lcd_home_menu.png)

| Key | Action |
|:---:|---|
| `A` | **Manual** mode |
| `B` | **Auto** mode |
| `C` | **Schedule** mode |
| `D` | **Settings** |
| `*` | Back to home (from any mode except alarm) |
| `#` | Confirm / **reset alarm** |

> In Proteus the keypad is labeled with math symbols. Mapping: `÷`=`A`, `×`=`B`, `−`=`C`, `+`=`D`, `=`=`*`, `ON/C`=`#`.

### Manual mode
`1` / `2` / `3` → turn on that zone (pump on) · `0` → all off.

### Auto mode
Waits until **soil is dry (RA0 = 1)** and **water is available (RA1 = 1)**, then runs **Zone 1 → Zone 2 → Zone 3**, repeats for the configured number of cycles, shows `DONE`, and goes back to idle.

### Schedule mode
Counts down `StartAfter` seconds (`WT` on the LCD), then starts the same irrigation sequence if the sensors allow it.

### Settings mode
| Key | Action |
|:---:|---|
| `1` / `2` / `3` | Edit duration **T1 / T2 / T3** of each zone |
| `4` | Edit number of **cycles** |
| `5` | Edit **start delay** (schedule mode) |
| `8` / `2` | Increase / decrease the value |
| `#` | Save the value |
| `*` | Back |

**Default values:**

| Parameter | Default | Range |
|---|:---:|:---:|
| T1 (zone 1 time, seconds) | 10 | 1–99 |
| T2 (zone 2 time, seconds) | 8 | 1–99 |
| T3 (zone 3 time, seconds) | 6 | 1–99 |
| Cycles | 2 | 1–9 |
| Start delay (seconds) | 15 | 1–99 |

### Alarm
| Cause | 7-segment code | LCD |
|---|:---:|---|
| Emergency switch pressed (RA2 = 1) | `01` | `EMERG #RESET` |
| Water lost during a run (RA1 = 0) | `02` | `WATER #RESET` |

The alarm is **latched**: all outputs are switched off and the system stays locked until `#` is pressed.

---

## 🧠 How it works (the deeper version)

### Software architecture
The firmware follows a layered, state-machine design:

| Layer | Responsibility |
|---|---|
| **Input layer** | Keypad scanning, sensors, emergency switch |
| **Decision layer** | Current mode and state transitions |
| **Execution layer** | Pump, zone outputs, timers |
| **Display layer** | LCD and 7-segment updates |

**Modes:** `HOME → MAN / AUTO / SCH / SET`, plus `ALM` (alarm) which has top priority.
**Irrigation states:** `IDLE → WAIT → Z1 → Z2 → Z3 → (repeat cycles) → DONE → IDLE`.

Because everything is state-based, tasks cannot overlap — for example, nothing can start while an alarm is active.

### Timing with Timer1 interrupt
- `T1CON = 0x31` (prescaler 1:8) with an 8 MHz crystal → Timer1 ticks at 250 kHz.
- Preload `0xF63C` → an interrupt every **10 ms**.
- Each interrupt: multiplexes one 7-segment digit (≈ 50 Hz per digit, no visible flicker), updates the blink flag every 0.5 s, and after 100 ticks raises `secFlag` (**1 second**).
- The main loop uses `secFlag` to decrement countdowns and advance the irrigation state machine, so timing is not blocked by LCD or keypad delays.

### Keypad scanning & debouncing
Columns (`RB0–RB3`) are driven low one at a time while the rows (`RB4–RB7`, pull-ups enabled) are read. A key is accepted on a **new press only** (`k != 0 && keyOld == 0`) and re-read after 20 ms to filter contact bounce.

### Safety logic (checked on every loop)
```c
if (EMERGENCY_SW == 1 && mode != M_ALM) alarmOn(A_EMG);

if (mode != M_ALM &&
    (runState == R_WAIT || runState == R_Z1 || runState == R_Z2 || runState == R_Z3) &&
    WATER_OK == 0) alarmOn(A_WAT);
```

### Notes on the current behavior
- The low-water alarm is checked only while an automatic/scheduled run is active (`WAIT`/`Z1`–`Z3`). In **manual mode** only the emergency switch triggers an alarm.
- `lcd.c` / `lcd.h` are included in the MPLAB project but **not used** — `newfile.c` has its own LCD routines (`lcd_*`). One of the two can be removed.
- Some comments in `newfile.c` lost their original characters (shown as `????`); they can be rewritten in English.
- The cycle counter is shown as a single digit on the LCD, which is why cycles are limited to 9.

---

## 📁 Repository structure

```
.
├── README.md
├── firmware/
│   ├── src/
│   │   ├── main.c            # (newfile.c) main program: modes, state machine, ISR, keypad
│   │   ├── lcd.c / lcd.h     # LCD driver (currently unused)
│   │   └── Makefile, nbproject/   # MPLAB X project files
│   └── FARM.X.production.hex # Ready-to-flash firmware
├── simulation/
│   └── smart_farm.pdsprj     # Proteus project
├── docs/
│   └── smart farm.pdf        # Project presentation
└── images/
```

> Tip: add a `.gitignore` to skip the generated `build/`, `dist/`, `debug/` folders and Proteus backups (`Project Backups/`, `*.workspace`). Keep only the final `.hex`.

---

## 🚀 Getting started

### Run the simulation (no hardware needed)
1. Open `simulation/smart_farm.pdsprj` in **Proteus**.
2. Double-click the PIC16F877A and load `firmware/FARM.X.production.hex` as the program file (clock: 8 MHz).
3. Press **Play**, then use the keypad and the sensor/emergency switches.

### Rebuild the firmware
1. Install **MPLAB X IDE** and the **XC8** compiler (project built with XC8 3.10).
2. Open the `FARM.X` project, select device **PIC16F877A**, and click **Build**.

### Configuration bits
```c
#pragma config FOSC = HS    // high-speed crystal
#pragma config WDTE = OFF   // watchdog off
#pragma config PWRTE = ON   // power-up timer on
#pragma config BOREN = ON   // brown-out reset on
#pragma config LVP  = OFF   // low-voltage programming off
```

---

## 🔭 Possible improvements
- Replace the simulated switches with real soil-moisture and water-level sensors (ADC).
- Drive a real pump and solenoid valves through relays/MOSFETs with flyback protection.
- Store the settings (T1–T3, cycles, delay) in EEPROM so they survive power loss.
- Add a real-time clock for time-of-day scheduling.

---

## 👥 Team

**Done by:**
- Bashar Al-Waseai
- Abdulrhman Al-Aidaros

**Supervisor:** Shahab Al-Ashen

---

## 📄 License

Add a license of your choice (e.g. [MIT](https://choosealicense.com/licenses/mit/)) by creating a `LICENSE` file in the repository root.
