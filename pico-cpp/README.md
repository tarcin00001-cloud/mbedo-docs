# Raspberry Pi Pico (C++) Projects Library

Welcome to the Raspberry Pi Pico (RP2040) C++ interpreted-mode project library. 

This directory contains step-by-step guides for building simulated circuits and running Arduino C++ code on the Pico using MbedO's interpreted runtime.

---

## 📌 Default Pin Mapping Reference

When designing circuits for the Raspberry Pi Pico in the MbedO designer, use these default pins to maintain compatibility with the example code:

| Component Pin | Pico Pin | GPIO Number | Description |
| --- | --- | --- | --- |
| Onboard LED | GP25 | 25 | Built-in LED on the Pico |
| Warning LED | GP15 | 15 | External digital output |
| Push Button | GP16 | 16 | Digital input (pull-up configured) |
| Active Buzzer | GP14 | 14 | PWM/Digital output tone |
| DHT22 SDA | GP12 | 12 | Single-wire sensor data |
| I2C LCD SDA | GP4 | 4 (I2C0 SDA) | I2C Data line |
| I2C LCD SCL | GP5 | 5 (I2C0 SCL) | I2C Clock line |
| Analog Potentiometer | GP26 | 26 (ADC0) | 12-bit Analog input |

---

## ⚠️ Interpreted Mode Code Rules

To ensure your code compiles and runs locally in the browser sandbox, verify that:
- **No loops** (`for`, `while`) are declared.
- **No array buffers** (`int buffer[10]`) are declared.
- **No helper functions** are declared outside `setup()` and `loop()`.
- Variables representing sensor values use alias mappings.
