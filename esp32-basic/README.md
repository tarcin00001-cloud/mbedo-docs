# ESP32 Basic (Non-IoT) Projects Library

Welcome to the ESP32 Basic (Non-IoT) project library.

This directory contains step-by-step guides for building simulated circuits and running offline C++ code (GPIO, ADC, PWM, SPI, and I2C) on the ESP32 using MbedO's interpreted runtime.

---

## 📌 Default Pin Mapping Reference

When designing circuits for the ESP32 in the MbedO designer, use these default pins to maintain compatibility with the example code:

| Component Pin | ESP32 Pin | GPIO Number | Description |
| --- | --- | --- | --- |
| Onboard LED | GPIO 2 | 2 | Built-in LED on the ESP32 |
| Warning LED | GPIO 4 | 4 | External digital output |
| Push Button | GPIO 5 | 5 | Digital input |
| Active Buzzer | GPIO 18 | 18 | PWM/Digital output tone |
| DHT22 SDA | GPIO 23 | 23 | Single-wire sensor data |
| I2C LCD SDA | GPIO 21 | 21 (I2C SDA) | I2C Data line |
| I2C LCD SCL | GPIO 22 | 22 (I2C SCL) | I2C Clock line |
| Analog Potentiometer | GPIO 34 | 34 (ADC1_CH6) | 12-bit Analog input |

---

## ⚠️ Interpreted Mode Code Rules

To ensure your code compiles and runs locally in the browser sandbox, verify that:
- **No loops** (`for`, `while`) are declared.
- **No array buffers** are declared.
- **No custom helper functions** are declared outside `setup()` and `loop()`.
- ESP32-specific hardware modules like `ledc` and `FreeRTOS` task macros are used according to default shims.
