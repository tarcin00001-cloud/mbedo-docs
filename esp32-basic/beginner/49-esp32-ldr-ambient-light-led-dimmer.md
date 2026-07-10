# 49 - ESP32 LDR Ambient Light LED Dimmer

Use an LDR to automatically dim an LED based on ambient light — the brighter the room, the dimmer the LED, and the darker the room, the brighter the LED.

## Goal
Learn how to combine analog input (LDR ADC read) with PWM output (LEDC) to create an automatic brightness compensation loop — the principle behind adaptive backlight in phones and laptop screens.

## What You Will Build
An LDR voltage divider on GPIO 34. The ADC reading is inverted and mapped to an 8-bit LEDC duty cycle on GPIO 5. The LED dims when the room is bright (LDR resistance is low, ADC is high) and brightens when the room is dark (LDR resistance is high, ADC is low).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LDR (photoresistor) | `ldr` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |
| LED (white or yellow) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR | Leg 1 | 3V3 | Red | Supply voltage (top of divider) |
| LDR | Leg 2 | GPIO34 | Yellow | Divider midpoint — ADC input |
| 10 kΩ Resistor | Leg 1 | GPIO34 | White | Pull-down leg of divider |
| 10 kΩ Resistor | Leg 2 | GND | Black | Ground reference |
| LED | Anode (+) | GPIO5 via 330 Ω | Orange | Brightness-controlled output |
| LED | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** The inversion logic is key: bright room → LDR resistance falls → ADC rises → `map()` inverts → PWM duty falls → LED dims. Dark room → LDR resistance rises → ADC falls → `map()` inverts → PWM duty rises → LED brightens. The 330 Ω resistor on the LED protects it even at full (255) PWM duty.

## Code
```cpp
// LDR Ambient Light LED Dimmer — automatic brightness compensation
const int LDR_PIN  = 34;
const int LED_PIN  = 5;
const int PWM_CHAN = 0;
const int PWM_FREQ = 5000;   // 5 kHz — flicker-free
const int PWM_RES  = 8;      // 8-bit: 0–255

void setup() {
  ledcSetup(PWM_CHAN, PWM_FREQ, PWM_RES);
  ledcAttachPin(LED_PIN, PWM_CHAN);

  Serial.begin(115200);
  Serial.println("Ambient Light Dimmer ready.");
}

void loop() {
  int raw = analogRead(LDR_PIN);   // 0 (dark) → 4095 (bright)

  // Invert: dark room → high duty (bright LED); bright room → low duty (dim LED)
  int duty = map(raw, 0, 4095, 255, 0);
  duty = constrain(duty, 0, 255);

  ledcWrite(PWM_CHAN, duty);

  Serial.print("LDR ADC: "); Serial.print(raw);
  Serial.print("  |  PWM duty: "); Serial.print(duty);
  Serial.print("  |  Brightness: ");
  Serial.print(duty * 100 / 255);
  Serial.println("%");

  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **LDR**, and **LED** onto the canvas.
2. Connect LDR **output** to **GPIO34**, LED **anode** to **GPIO5**.
3. Paste the code and click **Run**.
4. Drag the LDR slider toward "dark" — LED should brighten.
5. Drag the LDR slider toward "bright" — LED should dim.

## Expected Output
Serial Monitor:
```
Ambient Light Dimmer ready.
LDR ADC: 3800  |  PWM duty: 18   |  Brightness: 7%
LDR ADC: 2100  |  PWM duty: 124  |  Brightness: 48%
LDR ADC: 800   |  PWM duty: 206  |  Brightness: 80%
LDR ADC: 150   |  PWM duty: 246  |  Brightness: 96%
```

## Expected Canvas Behavior
* LED is nearly off when the LDR slider is set to maximum brightness (bright room).
* LED is at near-full brightness when the LDR slider is at minimum (dark room).
* LED brightness changes smoothly as the slider moves.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `ledcSetup(PWM_CHAN, PWM_FREQ, PWM_RES)` | Configures LEDC channel 0 at 5 kHz, 8-bit resolution. |
| `ledcAttachPin(LED_PIN, PWM_CHAN)` | Binds GPIO 5 to the LEDC channel. |
| `map(raw, 0, 4095, 255, 0)` | Inverted mapping: high ADC → low duty, low ADC → high duty. |
| `constrain(duty, 0, 255)` | Safety clamp prevents out-of-range PWM values. |
| `duty * 100 / 255` | Converts the 8-bit duty to a percentage for the Serial log. |

## Hardware & Safety Concept: Adaptive Backlight Control (ABC)
Every modern smartphone, laptop, and TV uses an **Automatic Brightness Control** loop identical to this project: a photosensor measures ambient light, and the display backlight PWM duty cycle is adjusted inversely — brighter surroundings need less backlight (the display is already washed out by ambient light), while darker surroundings need more (a dim backlight is unreadable in a dark room). This technique saves significant power: reducing backlight from 100% to 30% in a bright room can save 50–70% of display power. The same principle is applied in: street lights (brighten at night, dim at dawn), automotive dashboard illumination, emergency exit signs, and architectural accent lighting.

## Try This! (Challenges)
1. **Threshold dim**: Instead of continuous mapping, only turn the LED on above 50% brightness and off below — a hysteresis night-light variant.
2. **Log to percentage only**: Suppress raw ADC logging and print only the brightness percentage once per second for cleaner output.
3. **Three LEDs**: Use three LEDs (GPIOs 5, 12, 15) and light them one by one as darkness increases — a three-stage night light.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED always off | LDR reads very high (near 4095) in all conditions | Verify the pull-down resistor is between GPIO 34 and GND |
| LED brightness does not change | LDR wiper not connected to GPIO 34 | Re-seat the yellow wire |
| LED slightly flickering | PWM frequency too low | Increase `PWM_FREQ` to 10000 Hz |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [33 - ESP32 LDR Light Intensity Meter](33-esp32-ldr-light-intensity-meter.md)
- [34 - ESP32 LDR Automatic Dark Detector](34-esp32-ldr-automatic-dark-detector.md)
- [32 - ESP32 Potentiometer LED Dimmer](32-esp32-potentiometer-led-dimmer.md)
