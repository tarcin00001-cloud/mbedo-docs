# Raspberry Pi Pico (MicroPython) Projects Library

Welcome to the Raspberry Pi Pico (RP2040) MicroPython project library. 

This directory contains step-by-step guides for building simulated circuits and running Python code on the Pico using MbedO's interpreted MicroPython runtime.

---

## 📌 Default Pin Mapping Reference

When designing circuits for the Raspberry Pi Pico in the MbedO designer, use these default pins to maintain compatibility with the MicroPython examples:

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

## ⚠️ Interpreted MicroPython Rules

To ensure your code executes successfully inside the local Python interpreter, verify that:
- **No recursive functions** are declared.
- **Polling loops** contain short sleep cycles (`time.sleep()`) to prevent browser thread locking.
- Only the supported subset of the `machine` module (`Pin`, `ADC`, `PWM`, `I2C`) is used.
