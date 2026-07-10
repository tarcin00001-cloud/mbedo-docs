# 32 - ESP32 Potentiometer LED Dimmer

Use a potentiometer to continuously control the brightness of an LED through the ESP32's LEDC PWM hardware.

## Goal
Learn how to map a 12-bit ADC reading to an 8-bit PWM duty cycle using the ESP32's LEDC (LED Control) peripheral, producing smooth, flicker-free dimming.

## What You Will Build
A potentiometer wiper on GPIO 34 feeds the ADC. The raw value is mapped to a 0–255 PWM duty cycle driving an LED on GPIO 5 via the LEDC peripheral. Turning the knob continuously varies brightness from fully off to fully on.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Left leg (VCC end) | 3V3 | Red | Reference voltage |
| Potentiometer | Right leg (GND end) | GND | Black | Ground reference |
| Potentiometer | Wiper (middle leg) | GPIO34 | Yellow | ADC input |
| LED | Anode (+) | GPIO5 via 330 Ω | Orange | PWM dimming output |
| LED | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** GPIO 34 is an input-only ADC pin — do not try to drive it as an output. GPIO 5 is a full GPIO with PWM capability. Keep the 330 Ω resistor on the LED anode to limit current even at 100% duty cycle (full brightness).

## Code
```cpp
// Potentiometer LED Dimmer using ESP32 LEDC PWM
const int POT_PIN    = 34;   // ADC input
const int LED_PIN    = 5;    // PWM output
const int PWM_CHAN   = 0;    // LEDC channel (0–15)
const int PWM_FREQ   = 5000; // 5 kHz — above hearing, below ADC noise
const int PWM_RES    = 8;    // 8-bit resolution (0–255)

void setup() {
  // Configure LEDC PWM channel
  ledcSetup(PWM_CHAN, PWM_FREQ, PWM_RES);
  ledcAttachPin(LED_PIN, PWM_CHAN);

  Serial.begin(115200);
  Serial.println("Potentiometer LED Dimmer ready.");
}

void loop() {
  int rawADC = analogRead(POT_PIN);          // 0–4095 (12-bit)

  // Map 12-bit ADC range to 8-bit PWM duty cycle
  int dutyCycle = map(rawADC, 0, 4095, 0, 255);

  ledcWrite(PWM_CHAN, dutyCycle);            // Set PWM duty cycle

  Serial.print("ADC: "); Serial.print(rawADC);
  Serial.print("  |  PWM duty: "); Serial.println(dutyCycle);

  delay(50);   // ~20 Hz update rate for smooth response
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer**, and **LED** onto the canvas.
2. Connect Potentiometer **wiper** to **GPIO34**, LED **anode** to **GPIO5**.
3. Paste the code and click **Run**.
4. Drag the potentiometer slider — the LED brightness changes smoothly on the canvas.

## Expected Output
Serial Monitor:
```
Potentiometer LED Dimmer ready.
ADC: 0     |  PWM duty: 0
ADC: 512   |  PWM duty: 31
ADC: 2048  |  PWM duty: 127
ADC: 4095  |  PWM duty: 255
```

## Expected Canvas Behavior
* LED is off when the slider is fully left (ADC = 0, duty = 0).
* LED is at full brightness when the slider is fully right (ADC = 4095, duty = 255).
* LED brightness increases smoothly and continuously as the slider moves.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `ledcSetup(PWM_CHAN, PWM_FREQ, PWM_RES)` | Configures LEDC channel 0 at 5 kHz with 8-bit resolution. |
| `ledcAttachPin(LED_PIN, PWM_CHAN)` | Binds GPIO 5 to LEDC channel 0 for hardware PWM output. |
| `map(rawADC, 0, 4095, 0, 255)` | Linearly scales the 12-bit ADC range to the 8-bit PWM range. |
| `ledcWrite(PWM_CHAN, dutyCycle)` | Updates the hardware PWM duty cycle immediately. |
| `delay(50)` | 20 Hz update rate — smooth visual response without serial flooding. |

## Hardware & Safety Concept: LEDC Hardware PWM vs Software PWM
The ESP32's LEDC (LED Control) peripheral generates PWM signals entirely in hardware using dedicated timers. Unlike software PWM (which uses `digitalWrite` inside a delay loop), hardware PWM continues running even if the CPU is busy, does not stutter during Serial print operations, and produces a perfectly stable frequency. The LEDC peripheral has 16 channels (0–15), each assignable to any GPIO. It supports resolutions from 1-bit to 16-bit and frequencies from a few Hz to several MHz. For LED dimming at 5 kHz with 8-bit resolution, the minimum detectable brightness step is 3.3 V ÷ 255 ≈ **12.9 mV** — far too small for the human eye to see as a step, producing seamlessly smooth dimming.

## Try This! (Challenges)
1. **Logarithmic dimming**: The human eye perceives brightness logarithmically. Apply `dutyCycle = pow(2, rawADC / 512.0) - 1` for perceptually linear dimming.
2. **Dual LED**: Dim a second LED inversely — when the first is bright, the second is dim, and vice versa.
3. **Brightness percentage**: Print the brightness as a percentage (`dutyCycle * 100 / 255`) instead of a raw duty value.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays off or always on | Wrong PWM channel or pin attachment | Confirm `ledcAttachPin(LED_PIN, PWM_CHAN)` uses the same channel number as `ledcSetup` |
| Brightness jumps erratically | ADC noise on long wiper wire | Shorten the GPIO 34 wire; average 4–8 ADC readings before mapping |
| LED flickers visibly | PWM frequency too low | Increase `PWM_FREQ` from 5000 to 10000 Hz |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [31 - ESP32 Potentiometer ADC Read](31-esp32-potentiometer-adc-read.md)
- [33 - ESP32 LDR Light Intensity Meter](33-esp32-ldr-light-intensity-meter.md)
- [49 - ESP32 LDR Ambient Light LED Dimmer](49-esp32-ldr-ambient-light-led-dimmer.md)
