# Multi-Parameter Patient Monitoring System

**Project Topic:** Design and Development of a Multi-Parameter Patient Monitoring System with ECG Arrhythmia Detection, Real-Time Data Storage, and Wireless Remote Display.

This repository holds the hardware design (schematic, PCB, footprints, fabrication files) for the patient-side sensor and actuator unit. Firmware and the ML/logic layer on the Raspberry Pi are maintained separately by the software team; this repo covers circuitry only.

## What this device does

The board sits at the patient's bedside and handles all direct sensor and actuator interfacing:

- Reads ECG (AD8232), SpO2/pulse (MAX30102), body temperature (DS18B20), and blood pressure cuff pressure (MPXHZ6400).
- Drives a pump and a solenoid/valve for the blood pressure cuff inflate/deflate cycle.
- Displays local status on a 2.4" OLED and gives audible alerts through a buzzer.
- Streams sensor data wirelessly and receives commands over WiFi/MQTT.

The Raspberry Pi 5 sits on the nurse side, runs the ML model (arrhythmia detection), logic, data storage, and drives an HDMI display plus a web app. It never touches the patient directly, all sensor/actuator interfacing is the ESP32's job.

## System architecture (high level)

```
[Patient side]                              [Nurse side]
ESP32 (sensors + actuators + OLED + buzzer)  <--MQTT/WiFi-->  Raspberry Pi 5 (ML, logic, storage, HDMI, web app)
```

MQTT topic structure (for reference, implemented in firmware):

Pi publishes (commands to ESP32):
- `pi/control/pump`
- `pi/control/valve`
- `pi/display/oled`
- `pi/control/buzzer`

Pi subscribes (data from ESP32):
- `esp32/sensors/ecg`
- `esp32/sensors/bp`
- `esp32/sensors/temp`
- `esp32/sensors/pulse`
- `esp32/status/pump`

A full block diagram will be added to `docs/` covering both the electrical and system-level data flow.

## Repository structure

```
Multi-parameter Medical Measurement/
|-- libraries/
|     |-- 3d          (3D models for footprints, if used)
|     |-- footprints   (custom KiCad footprints, e.g. module outlines)
|     |-- symbols      (custom KiCad symbols for breakout modules)
|-- fabrication         (gerbers, drill files, BOM, pick-and-place, for JLCPCB)
|-- kicad                (project file, schematic, PCB layout)
|-- docs                 (this documentation, block diagrams, reference notes)
```

## Hardware summary

| Subsystem | Part | Notes |
|---|---|---|
| MCU | ESP32 DevKit V1 (30 GPIO) | Mounted on female header for clearance |
| ECG | AD8232 breakout | LO+/LO- leads-off detection wired in |
| SpO2 / Pulse | MAX30102 breakout | I2C |
| Temperature | DS18B20 | 1-Wire, 4.7k pull-up |
| Blood pressure | MPXHZ6400AC6T1 | SOIC-8, resistor divider scales output for ESP32 ADC |
| Display | 2.4" OLED, SPI, 7-pin | CS tied to GND (only SPI device on board) |
| Alert | Piezo buzzer | Driven through BC337 NPN transistor stage |
| Pump | 12V, MOSFET-switched | IRLB8721, flyback diode, gate pull-down |
| Solenoid/Valve | 12V, MOSFET-switched | IRLB8721, flyback diode, gate pull-down |
| Power | 3x 18650 (3S) + BMS | 12.6V nominal, LM2596 module steps down to 5V |

Full pin mapping and design rationale are in [`docs/circuit_documentation.md`](docs/circuit_documentation.md).

## Getting started

1. Open `kicad/Multi-parameter Medicals.kicad_pro` in KiCad 9.0.7 or later.
2. Custom symbols and footprints for breakout modules (AD8232, MAX30102, LM2596 module, BMS module) live under `libraries/`, make sure these are added to your project's library table before opening the schematic, or KiCad will show missing symbol/footprint errors.
3. The board is single-layer, routed with net ties and jumper wires (JP1-JP6, NT1-NT3) to work around single-layer constraints. If you're re-routing, keep the ground pour intact between the analog sensor section (left side) and the motor/switching section (right side), see the documentation for why.
4. Before sending to fab, run a full DRC pass and verify trace widths on the high-current nets (battery/BMS input, pump, solenoid) against actual component current ratings.

## Status / open items

- Block diagram (electrical + system-level): pending.
- Buzzer part number/current rating: confirm against datasheet once sourced (NPN driver stage is already in place regardless).
- Trace width verification on 12V/high-current nets: pending final pump/solenoid datasheet check.
- Mounting hole placement: confirm before final fab.

## Team

Circuit design (schematic + PCB): Ayodele, VECTAR ENERGY / INNOV8HUB
Firmware and ML/logic: student team

