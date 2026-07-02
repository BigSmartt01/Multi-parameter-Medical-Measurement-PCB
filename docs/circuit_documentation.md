# Circuit Documentation: Multi-Parameter Patient Monitoring System

This document explains the schematic and PCB design decisions for the patient-side hardware unit. It's meant as a reference for anyone working on this board, whether debugging, modifying, or just trying to understand why something was wired the way it was.

Project topic: Design and Development of a Multi-Parameter Patient Monitoring System with ECG Arrhythmia Detection, Real-Time Data Storage, and Wireless Remote Display.

---

## 1. System overview

The board is the patient-facing sensor and actuator unit. It does not run any ML logic, does not store data long-term, and does not drive the main display, all of that is handled by a Raspberry Pi 5 on the nurse side. The ESP32 on this board's only jobs are:

1. Read four physiological parameters (ECG, SpO2/pulse, temperature, blood pressure via cuff).
2. Control the pump and solenoid/valve for the BP cuff inflate/deflate cycle.
3. Show local status on an OLED and sound a buzzer for local alerts.
4. Talk to the Raspberry Pi wirelessly (WiFi/MQTT), sending sensor data out and receiving control commands in.

Communication between the ESP32 and the Raspberry Pi is entirely wireless, so it does not add any circuitry to this board, it's handled by the ESP32's onboard WiFi.

---

## 2. Power architecture

```
3x 18650 (series) --> 3S BMS --> Push Button Switch --> [12V rail]
                                                            |
                                                            +--> Pump / Solenoid (via MOSFETs)
                                                            |
                                                            +--> Pump LED, Solenoid LED
                                                            |
                                                            +--> LM2596 buck module --> [5V rail]
                                                                                          |
                                                                                          +--> ESP32 VIN
                                                                                          +--> MPXHZ6400 pressure sensor
                                                                                          +--> Power LED

ESP32 onboard 3.3V LDO --> [3V3 rail] --> AD8232, MAX30102, DS18B20, OLED, Buzzer
```

**Why this split:** The pump and solenoid are the highest current draw items on the board and run directly off 12V rather than through the buck converter, this keeps the LM2596 sized only for what the 5V rail actually needs (ESP32 + pressure sensor), and avoids putting motor switching transients through the same regulator that's trying to hold a clean 5V for an analog sensor.

**BMS:** 3S configuration matches the three series 18650 cells, handles balancing and over-discharge/overcurrent protection.

**Bulk capacitance:** A 1000uF capacitor (C2) sits at the LM2596 output to absorb transients, particularly important given the pump/solenoid switching nearby can cause momentary sag on the input side that would otherwise ripple through to the 5V rail.

---

## 3. Sensor interfacing

### 3.1 ECG (AD8232) — J2

| Pin | Net | ESP32 GPIO |
|---|---|---|
| 3V3 | 3V3 | — |
| ECG_OUT | ECG signal | GPIO34 (ADC1_CH6, input-only) |
| LO- | Leads-off negative | GPIO33 |
| LO+ | Leads-off positive | GPIO32 |
| GND | GND | — |

A 100nF decoupling capacitor (C3) sits at the 3V3 supply pin of J2, added specifically because this board also carries motor-switching circuitry, the cap is local insurance against supply noise reaching the ECG front end.

**LO+/LO- matter.** These are digital outputs from the AD8232 that go HIGH when an electrode has detached from the skin. Without reading them, a loose lead produces garbage or flatline data indistinguishable from an actual reading, both for the arrhythmia detection model and for anyone monitoring the signal. They're read as simple digital HIGH/LOW inputs in firmware.

### 3.2 SpO2 / Pulse (MAX30102) — J3

I2C device, shares the bus with nothing else on this board (OLED is SPI, not I2C).

| Pin | Net | ESP32 GPIO |
|---|---|---|
| 3V3 | 3V3 | — |
| SDA | I2C data | GPIO21 |
| SCL | I2C clock | GPIO22 |
| GND | GND | — |

A 100nF decoupling cap (C4) is present at the 3V3 pin, same rationale as the ECG module.

### 3.3 Temperature (DS18B20) — J4

1-Wire device.

| Pin | Net | ESP32 GPIO |
|---|---|---|
| 3V3 | 3V3 | — |
| TEMP | Data | GPIO4 |
| GND | GND | — |

R5 (4.7k) is a pull-up resistor between 3V3 and the TEMP data line, required by the 1-Wire protocol.

### 3.4 Blood pressure / cuff pressure (MPXHZ6400AC6T1) — IC1

This is a bare SOIC-8 sensor IC, not a breakout module, so it needs its own supply and signal conditioning on this board.

**Pinout note:** only VS (supply), GND, and VOUT are connected. Pins DNC_1 through DNC_5 (5 of the 8 pins) are internal device connections per the datasheet and must not be connected to anything, including ground.

**Signal conditioning:** the sensor's raw output swings up to roughly 4.7V at full scale (5V supply), which exceeds the ESP32 ADC's 3.3V input limit. A resistor divider scales this down before it reaches GPIO35:

```
VOUT --[R1: 6.8k]--+--[R2: 12k]-- GND
                    |
                    +-- C1 (100nF) -- GND
                    |
                    BP_OUT --> GPIO35 (ESP32 ADC, input-only)
```

At the sensor's absolute worst-case output (4.7V), this divider brings the signal down to roughly 3.0V, safely under the ADC limit even in an overpressure fault condition. The 100nF capacitor (C1) filters switching noise picked up on the line before it reaches the ADC, added specifically because of the pump/solenoid switching happening elsewhere on the board.

**Assembly note:** this is the only SMD component on an otherwise all-through-hole board. It's SOIC-8 package with 1.27mm pin pitch, hand-solderable with a normal fine-tip iron. J12 (BP_JST) is included as a fallback in case the team decides to source a pre-assembled breakout module instead of hand-soldering the bare IC, either path works with this board.

---

## 4. Actuator interfacing (Pump and Solenoid/Valve)

Both follow an identical MOSFET low-side switching pattern.

```
GPIO --[220R gate resistor]-- MOSFET Gate
                               |
GND --[10k pull-down]---------+
                               
MOSFET Drain -- Load (Pump or Solenoid) -- 12V
MOSFET Source -- GND

Flyback diode: cathode to 12V, anode to switched node (across the load)
```

| Actuator | MOSFET | Gate GPIO | Gate resistor | Pull-down | Flyback diode |
|---|---|---|---|---|---|
| Pump | Q1 (IRLB8721) | GPIO26 | R3 (220R) | R7 (10k) | D1 (1N4007) |
| Solenoid/Valve | Q2 (IRLB8721) | GPIO27 | R4 (220R) | R6 (10k) | D2 (1N4007) |

**Why IRLB8721:** logic-level MOSFET, fully enhanced well under 3.3V gate drive, so it switches cleanly straight off an ESP32 GPIO through the gate resistor without needing a separate gate driver IC.

**Why the gate pull-down (10k):** during ESP32 boot/reset, GPIOs can float momentarily. Without the pull-down, a floating gate could let the MOSFET partially conduct unintentionally. The pull-down holds the gate at GND (MOSFET off) until firmware actively drives it high.

**Why the flyback diode:** both the pump and solenoid are inductive loads. When the MOSFET switches off, the collapsing magnetic field generates a voltage spike that can exceed the MOSFET's rated drain-source voltage and destroy it. The flyback diode gives that current a safe path to circulate (cathode to 12V, anode to the switched node) rather than spiking through the MOSFET.

**Indicator LEDs:** D4 (pump) and D5 (solenoid) are wired in parallel across each load's switched node, so they light whenever that actuator is energized, no separate GPIO or logic needed. 1k current-limiting resistors (R9, R10) since these run off the 12V rail.

---

## 5. Display and alert

### 5.1 OLED (2.4", SPI, 7-pin) — J11

| Pin | Net | ESP32 GPIO |
|---|---|---|
| 3V3 | 3V3 | — |
| D0 | SPI Clock | GPIO18 |
| D1 | SPI MOSI | GPIO23 |
| RES | Reset | GPIO16 |
| DC | Data/Command | GPIO17 |
| CS | Chip Select | Tied to GND |
| GND | GND | — |

**Note on CS:** normally CS would be driven by a GPIO (GPIO5 in early revisions of this design), but since this OLED is the only SPI device on the board, CS is tied permanently to GND instead. This is standard practice for single-device SPI buses, it saves a GPIO and works because there's no bus contention to arbitrate.

### 5.2 Buzzer — J10

The buzzer is not driven directly off a GPIO. GPIO pins can't reliably source enough current for most buzzers, so it goes through an NPN transistor low-side switch:

```
3V3 -- Buzzer(+)
Buzzer(-) -- Q3 Collector (BC337)
GPIO25 --[R11: 1k]-- Q3 Base
Q3 Emitter -- GND
```

GPIO25 through the 1k base resistor is enough to saturate the BC337 and switch the buzzer on/off cleanly, without stressing the GPIO pin regardless of the buzzer's actual current draw.

---

## 6. Complete GPIO map

| GPIO | Function | Notes |
|---|---|---|
| 4 | DS18B20 temperature data | 4.7k pull-up (R5) |
| 5 | (unused — CS net grounded instead) | See OLED section |
| 16 | OLED RES | |
| 17 | OLED DC | |
| 18 | OLED D0 / SPI CLK | |
| 21 | MAX30102 SDA | I2C |
| 22 | MAX30102 SCL | I2C |
| 23 | OLED D1 / SPI MOSI | |
| 25 | Buzzer (via Q3 base) | |
| 26 | Pump gate | via R3/R7 |
| 27 | Solenoid/Valve gate | via R4/R6 |
| 32 | AD8232 LO+ | |
| 33 | AD8232 LO- | |
| 34 | AD8232 ECG output | ADC1, input-only |
| 35 | Blood pressure sensor output | ADC1, input-only, via divider |

9 functional GPIOs in active use, well within the ESP32's available pin count, leaving room for future additions.

---

## 7. PCB layout notes

### 7.1 Single-layer constraint

This board is routed on a single copper layer, using jumper wires and net ties (JP1-JP6, NT1-NT3) to bridge connections that would otherwise require a second layer. This is a deliberate cost/complexity tradeoff for a student-fabricated board, but it means routing discipline matters more than it would on a 2-layer design, particularly around ground continuity.

### 7.2 Placement zones

The board is organized into two zones kept physically separated:

- **Sensor/signal side (left):** AD8232, MAX30102, DS18B20, OLED, and their associated JST connectors.
- **Power/switching side (right/bottom):** MOSFETs, flyback diodes, gate resistors, LM2596, BMS input, screw terminals for pump/solenoid.

This separation exists to keep switching noise from the pump/solenoid MOSFETs away from the sensitive analog ECG and pulse signals. The AD8232 connector (J2) in particular was originally placed too close to the MOSFET cluster during early layout and was moved as part of the design review, keep this in mind if the board is ever re-laid-out, that adjacency is the single most noise-sensitive placement decision on the board.

### 7.3 Ground pour

A single ground pour is used across the board (not split analog/digital grounds). This works because of placement discipline rather than pour topology: high-current return paths (motor switching) are kept geometrically away from the direct line between sensor connector grounds and the main ground return, so a shared pour doesn't inject switching noise into the sensor grounds under normal operation. If ECG readings show noise correlated with pump/solenoid activity during testing, the fix is more likely a ferrite bead on the AD8232 supply line than a ground pour rework.

### 7.4 ESP32 antenna keepout

The ESP32 DevKit module has a WiFi antenna area that needs to stay clear of copper pour, traces, and components, both on this board and in whatever enclosure houses it. Check the specific antenna zone on the module's silkscreen/datasheet and maintain that keepout, this matters more here than on a typical project since the board relies on continuous bidirectional WiFi/MQTT communication.

### 7.5 Before fabrication

- Run a full DRC pass, the jumper/net-tie routing is dense in places and worth double-checking for accidental bridges or unintended opens.
- Verify trace widths on high-current nets (battery/BMS path, pump, solenoid) against the actual rated current of the pump and solenoid being used, not just an assumed default.
- Confirm mounting hole placement against the enclosure design.
- Do a 3D/clearance check, especially around densely packed clusters, to make sure hand soldering and wire access to screw terminals is realistic.

---

## 8. Bill of Materials highlights

Full BOM is exported from KiCad under `fabrication/`. Key parts requiring careful sourcing:

| Part | Why it matters |
|---|---|
| MPXHZ6400AC6T1 | SOIC-8, only 3 of 8 pins connected, verify pinout against datasheet chamfer/pin-1 marking before soldering |
| IRLB8721 (x2) | Logic-level MOSFET, confirm TO-220 pinout (G-D-S) matches datasheet, pinout can vary by manufacturer even within the same package |
| 1N4007 (x2) | Flyback diodes, orientation is safety-critical: cathode to 12V, anode to switched node |
| BC337 | Buzzer driver transistor |
| LM2596 module | Assumed to be the ready-made module (has onboard Schottky diode and inductor), not the bare IC |

---

## 9. Revision notes

Rev 1.0, dated 2026-07-01. Diode orientation, buzzer driver, and MOSFET/AD8232 placement were reviewed and corrected during design review prior to this revision. See open items in the repository README for outstanding checks before fabrication.
