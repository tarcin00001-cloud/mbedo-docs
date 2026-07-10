# Beginner Projects

29 projects using 1–2 components and pure GPIO / analog patterns.
No library knowledge required beyond `Serial.begin`.

---

## What You Need

All beginner projects use the **Arduino Uno** board (`uno`).
You do not need any advanced libraries — the interpreter handles everything built-in.

---

## Batch 1A — Output

Learn to control LEDs, buzzers, and RGB outputs.

| # | Project | Components | Key Skill |
| --- | --- | --- | --- |
| 01 | [LED Blink](01-led-blink.md) | LED | `digitalWrite` + `delay` |
| 02 | [LED Fade](02-led-fade.md) | LED | `analogWrite` PWM |
| 03 | [Breathing LED](03-breathing-led.md) | LED | Step-wise PWM |
| 04 | [RGB Color Mixer](04-rgb-color-mixer.md) | RGB LED | Three PWM channels |
| 05 | [Traffic Light Sequence](05-traffic-light-sequence.md) | Traffic Light | Multi-output sequencing |
| 06 | [WS2812B Color Cycle](06-ws2812b-color-cycle.md) | WS2812B LED | Adafruit NeoPixel |
| 07 | [Buzzer Beep](07-buzzer-beep.md) | Buzzer | `tone()` + `noTone()` |
| 08 | [Buzzer Melody](08-buzzer-melody.md) | Buzzer | Frequency sequence |
| 09 | [Doorbell Tone](09-doorbell-tone.md) | Push Button + Buzzer | Input → sound |
| 10 | [Alarm Oscillate](10-alarm-oscillate.md) | Buzzer | Alternating frequencies |

## Batch 1B — Input

Read buttons, potentiometers, and touch sensors.

| # | Project | Components | Key Skill |
| --- | --- | --- | --- |
| 11 | [Button ON/OFF](11-button-onoff.md) | Push Button + LED | `digitalRead` |
| 12 | [Button Toggle](12-button-toggle.md) | Push Button + LED | State variable |
| 13 | [Button Counter Print](13-button-counter-print.md) | Push Button | Serial counter |
| 14 | [Dial LED Brightness](14-dial-led-brightness.md) | Potentiometer + LED | `analogRead` → `map` → `analogWrite` |
| 15 | [Analog Meter Serial](15-analog-meter-serial.md) | Potentiometer | `analogRead` → Serial |
| 16 | [Touch ON/OFF](16-touch-onoff.md) | TTP223 Touch + LED | Capacitive input |
| 17 | [Hold-to-Light](17-hold-to-light.md) | Push Button + LED | Hold detection |

## Batch 1C — Sensors

Read environment sensors and print values to Serial Monitor.

| # | Project | Components | Key Skill |
| --- | --- | --- | --- |
| 18 | [Light Meter](18-light-meter.md) | LDR | Analog light reading |
| 19 | [Night Detector](19-night-detector.md) | LDR + LED | Threshold comparison |
| 20 | [Thermistor Raw Print](20-thermistor-raw-print.md) | NTC Thermistor | Analog temperature |
| 21 | [Motion Alert Print](21-motion-alert-print.md) | PIR | Digital motion |
| 22 | [Obstacle Print](22-obstacle-print.md) | IR Sensor | Digital obstacle |
| 23 | [Knock Detector](23-knock-detector.md) | Piezo Sensor + LED | Vibration threshold |
| 24 | [Rain Alert](24-rain-alert.md) | Rain Sensor + LED | Digital rain signal |
| 25 | [Rain Level Print](25-rain-level-print.md) | Rain Sensor | Analog rain level |
| 26 | [Water Level Print](26-water-level-print.md) | Water Level Sensor | Analog water level |
| 27 | [Touch Lamp](27-touch-lamp.md) | TTP223 Touch + LED | Capacitive control |
| 28 | [Flame Detected Print](28-flame-detected-print.md) | Flame Sensor | Digital flame |
| 29 | [Gas Level Print](29-gas-level-print.md) | MQ-2 Gas | Analog gas level |
