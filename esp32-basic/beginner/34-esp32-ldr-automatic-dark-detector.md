# 34 - ESP32 LDR Automatic Dark Detector

Automatically switch an LED on when ambient light drops below a set threshold — a software-controlled automatic night light.

## Goal
Learn how to apply a threshold comparison to an analog sensor reading and drive a digital output based on whether the environment is dark or light.

## What You Will Build
An LDR voltage divider on GPIO 34. When the ADC reading falls below a darkness threshold (light level low), an LED on GPIO 5 turns on automatically. When light returns above the threshold, the LED turns off.

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
| LDR | Leg 1 | 3V3 | Red | Supply voltage |
| LDR | Leg 2 | GPIO34 | Yellow | Voltage divider midpoint |
| 10 kΩ Resistor | Leg 1 | GPIO34 | White | Pull-down leg of divider |
| 10 kΩ Resistor | Leg 2 | GND | Black | Ground reference |
| LED | Anode (+) | GPIO5 via 330 Ω | Orange | Automatic night light |
| LED | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** The ADC reads a low voltage when the room is dark (high LDR resistance). The threshold constant in the code (default 800) determines the changeover point — calibrate it by reading the ADC value at your target "it's dark" light level from Project 33 and using that value as your threshold.

## Code
```cpp
// LDR Automatic Dark Detector — Night Light
const int LDR_PIN   = 34;
const int LED_PIN   = 5;
const int THRESHOLD = 800;    // ADC < 800 = dark → LED on (calibrate as needed)

void setup() {
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  Serial.begin(115200);
  Serial.println("Auto Dark Detector ready.");
}

void loop() {
  int ldrValue = analogRead(LDR_PIN);

  if (ldrValue < THRESHOLD) {
    // Dark — turn night light on
    digitalWrite(LED_PIN, HIGH);
    Serial.print("DARK  — LED ON  | ADC: ");
  } else {
    // Light — turn night light off
    digitalWrite(LED_PIN, LOW);
    Serial.print("LIGHT — LED OFF | ADC: ");
  }

  Serial.println(ldrValue);
  delay(300);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **LDR**, and **LED** onto the canvas.
2. Connect LDR **output** to **GPIO34**, LED **anode** to **GPIO5**.
3. Paste the code and click **Run**.
4. Move the LDR slider below the threshold to simulate darkness — LED turns on.
5. Move the slider above the threshold to simulate daylight — LED turns off.

## Expected Output
Serial Monitor:
```
Auto Dark Detector ready.
LIGHT — LED OFF | ADC: 3200
LIGHT — LED OFF | ADC: 2900
DARK  — LED ON  | ADC: 650
DARK  — LED ON  | ADC: 120
LIGHT — LED OFF | ADC: 1800
```

## Expected Canvas Behavior
* LED is off while the LDR slider is in the bright region.
* LED turns on automatically when the slider crosses below the threshold.
* Transition is immediate on the next 300 ms poll cycle.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `const int THRESHOLD = 800` | The ADC value below which the room is considered dark. Adjust to calibrate. |
| `if (ldrValue < THRESHOLD)` | Compares the live ADC reading against the darkness threshold. |
| `digitalWrite(LED_PIN, HIGH)` | Turns on the night light when dark. |
| `digitalWrite(LED_PIN, LOW)` | Turns off the night light when light is sufficient. |

## Hardware & Safety Concept: Threshold-Based Automation
A threshold is a decision boundary that divides a continuous measurement into two actionable states (ON/OFF, ALERT/NORMAL, OPEN/CLOSE). Almost all real-world automatic systems use threshold logic: a thermostat turns on heating below 18 °C and off above 20 °C; a battery charger stops charging above 4.2 V; a street lamp turns on below a lux threshold. The threshold value is typically derived by **calibration** — measuring the sensor output under the exact target conditions and setting the boundary slightly above or below. A **hysteresis band** (different on-threshold and off-threshold) is added in professional designs to prevent rapid oscillation near the boundary — see the Try This section below.

## Try This! (Challenges)
1. **Hysteresis**: Add a second threshold — turn on below 700 ADC counts, turn off above 1000 ADC counts — to prevent the LED flickering at the boundary.
2. **Adjustable threshold**: Read a potentiometer on GPIO 35 and use its mapped value as the `THRESHOLD`, making it user-adjustable without recompiling.
3. **Delayed switch**: Only turn the LED on if the ADC has been below the threshold for 3 consecutive seconds (debounce for light).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED never turns on even in darkness | Threshold too low — LDR in darkness reads above it | Read the actual dark ADC value from Project 33 and set threshold just above it |
| LED never turns off in daylight | Threshold too high | Reduce `THRESHOLD` so that normal room lighting ADC value exceeds it |
| LED flickers near threshold | ADC noise at the boundary | Implement hysteresis (Challenge 1 above) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [33 - ESP32 LDR Light Intensity Meter](33-esp32-ldr-light-intensity-meter.md)
- [49 - ESP32 LDR Ambient Light LED Dimmer](49-esp32-ldr-ambient-light-led-dimmer.md)
- [36 - ESP32 PIR Motion Sensor Alert](36-esp32-pir-motion-sensor-alert.md)
