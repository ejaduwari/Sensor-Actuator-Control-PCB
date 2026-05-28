# Sensor-Actuator-Control-PCB
STM32F103C8 carrier board controlling 6 stepper motors, servos, limit switches, and IO expanders with UART, SPI, and I2C interfaces. Includes full pin mapping, dual-regulator power design, and detailed failure analysis with debugging insights and hardware lessons learned.

# 📘 STM32F103C8 Stepper Motor Controller  
### Carrier Board + Small Pill Architecture & Debug Report

---

## 📑 Table of Contents
- [Overview](#overview)
- [Hardware Architecture](#hardware-architecture)
- [Pin Mapping (STM32 ↔ Carrier Board)](#pin-mapping-stm32--carrier-board)
- [Peripheral Interfaces](#peripheral-interfaces)
- [Clock & Crystal](#clock--crystal)
- [Power System Design](#power-system-design)
- [IO Expanders](#io-expanders)
- [Failure Report & Debugging](#failure-report--debugging)
- [Root Cause Analysis](#root-cause-analysis)
- [Current Status](#current-status)
- [Design Lessons & Prevention](#design-lessons--prevention)

---

## 🔍 Overview

This project is a **custom STM32F103C8-based PCB** designed to control:
- Multiple **A4988 stepper drivers**
- Several **servo outputs**
- External IO via expanders

The board uses a **Small Pill (STM32F103C8T6)** mounted on a **carrier board**, providing structured routing for motion control and expansion.

---

## 🧩 Hardware Architecture

- **MCU:** STM32F103C8T6 (Small Pill)
- **Motor Drivers:** A4988 (external)
- **IO Expansion:** PI4IOE5V6416
- **Power Inputs:**
  - USB (via L78L00 regulator)
  - External 5V (via AP2112K-3.3)
- **Control Signals:**
  - STEP / DIR for 6 axes
  - Servo PWM outputs
- **Debug Interface:**
  - SWD (SWCLK, SWDIO)

---

## 🔌 Pin Mapping (STM32 ↔ Carrier Board)

| STM32 Pin | Function |
|----------|--------|
| PA0 | 5_DIR |
| PA1 | 5_STP |
| PA2 | TX |
| PA3 | RX |
| PA4 | 1_DIR |
| PA5 | SCK |
| PA6 | MISO |
| PA7 | MOSI |
| PA8 | SERVO_1_3V3 |
| PA9 | SERVO_2_3V3 |
| PA10 | SERVO_3_3V3 |
| PA11 | USB- |
| PA12 | USB+ |
| PA15 | SERVO_4_3V3 |

| STM32 Pin | Function |
|----------|--------|
| PB0 | 4_STP |
| PB1 | 4_DIR |
| PB2 | BOOT |
| PB3 | 3_STP |
| PB4 | SERVO_6_3V3 |
| PB5 | SW9 |
| PB6 | SCL |
| PB7 | SDA |
| PB8 | 6_DIR |
| PB9 | SW10 |
| PB10 | 3_DIR |
| PB11 | 6_STP |
| PB12 | LED |
| PB13 | GPIO |
| PB14 | GPIO |
| PB15 | GPIO |

| STM32 Pin | Function |
|----------|--------|
| PC13 | 2_DIR |
| PC14 | 2_STP |
| PC15 | 1_STP |

| Debug Pins | Function |
|-----------|--------|
| PA13 | SWDIO |
| PA14 | SWCLK |
| NRST | Reset |

---

## 🔗 Peripheral Interfaces

### UART
- TX: PA2  
- RX: PA3  

### SPI
- SCK: PA5  
- MISO: PA6  
- MOSI: PA7  

### I2C
- SCL: PB6  
- SDA: PB7  

---

## ⏱ Clock & Crystal

- **Component:** 25 MHz SMD Crystal  
- **Capacitance:** 20pF  
- **Package:** 3.2mm × 2.5mm  

- **DigiKey Part #:** 3155-25M20P2/XT324-10/10CT-ND  
- **Manufacturer #:** 25M20P2/XT324-10/10  

---

## ⚡ Power System Design

### Regulators

**1. L78L00**
- Input: USB (~5V)
- Output: 3.3V (intended)
- Status: ❌ Damaged / unstable

**2. AP2112K-3.3**
- Input: External 5V
- Output: 3.3V
- Status: ✅ Functional

---

### Observed Behavior

- With motors connected:
  - 3.3V rail dropped to ~3.0V
- With L78L00 removed:
  - Stable 3.29V from AP2112K
- Backfeeding observed across regulators

---

## 🔌 IO Expanders

- **Model:** PI4IOE5V6416  
- Used for:
  - Microstepping control
  - Enable signals

⚠️ These lines interface with high-current drivers → risk area

---

## 🧪 Failure Report & Debugging

### Initial Symptoms

- PB12 LED stopped working
- MCU still flashable (initially)
- Reset did not recover functionality

---

### Voltage Measurements

| Condition | Observation |
|----------|------------|
| Motors connected | 3.3V ~3.0V |
| PB12 | ~0.12V |
| After disconnecting 12V | PB12 still ~0.12V |
| Reset held | 3.3V rises slightly |

---

### GPIO Testing

- PB12–PB15 stuck at ~0.12V
- PB15 toggle test failed
- Same issue after MCU replacement

---

### Regulator Testing

- L78L00:
  - Input ~4.8V
  - Output unstable / low
- AP2112K:
  - Output stable at ~3.29V

---

### Fuse

- 0685H9100-01 → OK (~0Ω)

---

### After L78L00 Removal

- System powered via AP2112K
- 3.3V stable
- ❌ Flashing failed:
  - Error in final launch sequence
  - Failed to start GDB server
 

---

## 🔎 Root Cause Analysis

### 1. GPIO Damage
- PB12–PB15 permanently stuck low
- Cause: **Backfeed / overvoltage from stepper drivers**
- Likely triggered by:
- Swapping motor wires while powered

---

### 2. Power Rail Instability
- L78L00 regulator damaged
- Caused rail drops under load

---

### 3. Regulator Conflict
- Backfeeding between regulators
- L78L00 became ineffective

---

### 4. Flashing Failure
- Likely due to:
- Unstable VDD reference
- Improper SWD voltage levels

---

## 📉 Current Status

- ❌ Original MCU damaged (GPIO bank failure)
- ⚠️ Replacement MCU still showing abnormal GPIO behavior
- ✅ AP2112K regulator working
- ❌ Board cannot be flashed
- ❌ System non-functional

---

## 🛠 Design Lessons & Prevention

### Critical Rules

- ❌ Never swap stepper wires under power
- ❌ Avoid direct MCU exposure to driver signals

---

### Recommended Fixes

- Add **series resistors** on control lines
- Use **optocouplers or buffers**
- Isolate **high-current paths**
- Ensure **single clean 3.3V source**
- Avoid regulator backfeeding

---

### Debug Best Practices

- Verify rails before applying load
- Test MCU pins before connecting drivers
- Use current limiting during bring-up

---

## 🚧 Final Note

This failure highlights a critical embedded design principle:

> **Power integrity and signal protection are just as important as logic design.**

Future revisions should prioritize:
- Electrical isolation
- Robust power architecture
- Safer bring-up procedures

---
