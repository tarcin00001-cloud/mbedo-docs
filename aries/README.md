# VEGA ARIES v3 (RISC-V) Projects Library

Welcome to the VEGA ARIES v3 project library.

This directory contains step-by-step guides for building simulated circuits and running C++ / assembly code on the VEGA ARIES board (based on the THEJAS32 RISC-V SoC) using MbedO's interpreted runtime.

---

## 📌 Default Pin Mapping Reference

When designing circuits for the VEGA ARIES board in the MbedO designer, use these default pins to maintain compatibility with the example code:

| Component Pin | ARIES Pin | Description |
| --- | --- | --- |
| Onboard LED (Red) | LED_R (GPIO 23) | Built-in RGB LED (Red) |
| Onboard LED (Green) | LED_G (GPIO 24) | Built-in RGB LED (Green) |
| Onboard LED (Blue) | LED_B (GPIO 25) | Built-in RGB LED (Blue) |
| Warning LED | GPIO 15 | External digital output |
| Push Button | GPIO 16 | Digital input (pull-up configured) |
| Active Buzzer | GPIO 14 | PWM/Digital output tone |
| DHT22 SDA | GPIO 12 | Single-wire sensor data |
| I2C LCD SDA | I2C0 SDA (A4) | I2C Data line |
| I2C LCD SCL | I2C0 SCL (A5) | I2C Clock line |
| Analog Potentiometer | ADC0 (GP26) | Analog input |

---

## ⚠️ Interpreted Mode Code Rules

To ensure your code compiles and runs locally in the browser sandbox, verify that:
- **No loops** (`for`, `while`) are declared.
- **No array buffers** are declared.
- **No custom helper functions** are declared outside `setup()` and `loop()`.
- Register-level variables and RISC-V assembly blocks are formatted according to the THEJAS32 register configuration standards.
