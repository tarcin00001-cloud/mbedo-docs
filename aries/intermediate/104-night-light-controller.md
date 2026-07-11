# 104 - Night Light Controller

Build an energy-efficient smart night light controller that activates a light relay only when ambient lighting is dark and human motion is detected.

## Goal
Learn how to create multi-sensor logical conditions (AND logic) using an analog photoresistor (LDR) and a digital PIR motion sensor to control a relay switch.

## What You Will Build
A smart corridor light. During daylight hours (high LDR voltage), the light relay remains OFF regardless of movement. At night (low LDR voltage), the light relay automatically switches ON as soon as the PIR sensor registers movement, turning off when the visitor departs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Photoresistor (LDR) | `ldr` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| 5V Relay Module (Light) | `relay` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR Circuit | VCC | 3V3 | Red | Voltage divider power |
| LDR Circuit | OUT (Signal) | ADC1 (GP27) | White | Analog light level output |
| LDR Circuit | GND | GND | Black | Ground reference |
| PIR Sensor | VCC | 3V3 | Red | PIR sensor operating power |
| PIR Sensor | GND | GND | Black | Ground reference |
| PIR Sensor | OUT | GPIO 17 | Green | Digital movement signal |
| Relay Module | VCC | 5V | Red | Relay coil operating voltage |
| Relay Module | GND | GND | Black | Ground reference |
| Relay Module | IN | GPIO 15 | Blue | Control signal pin |

> **Wiring tip:** The photoresistor requires a voltage divider circuit. Wire a 10k-ohm resistor from GND to ADC1 (GP27), and connect the LDR between 3V3 and ADC1 (GP27).

## Code
```cpp
const int LDR_PIN = 27;     // LDR on ADC1 (GP27)
const int PIR_PIN = 17;     // PIR on GPIO 17
const int RELAY_PIN = 15;   // Night Light Relay on GPIO 15

// Threshold value for darkness (LDR outputs lower voltage in dark)
const int DARK_THRESHOLD = 1500; 

void setup() {
  pinMode(PIR_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  
  // Ensure relay starts in off state
  digitalWrite(RELAY_PIN, LOW);
}

void loop() {
  int ldrValue = analogRead(LDR_PIN);
  int motionDetected = digitalRead(PIR_PIN);

  // Turn ON light only if it is dark AND motion is active
  if (ldrValue < DARK_THRESHOLD && motionDetected == HIGH) {
    digitalWrite(RELAY_PIN, HIGH);
  } else {
    digitalWrite(RELAY_PIN, LOW);
  }

  delay(200); // Polling delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Photoresistor (LDR)**, **PIR Motion Sensor**, and **5V Relay Module** components onto the canvas.
2. Wire the LDR Circuit: **VCC** to **3V3**, **OUT** to **ADC1 (GP27)**, and **GND** to **GND**.
3. Wire the PIR Sensor: **VCC** to **3V3**, **GND** to **GND**, and **OUT** to **GPIO 17**.
4. Wire the Relay: **VCC** to **5V**, **GND** to **GND**, and **IN** to **GPIO 15**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
LDR: 3200 (Light) | PIR: 0 (No Motion) -> Relay: OFF
LDR: 800 (Dark)   | PIR: 0 (No Motion) -> Relay: OFF
LDR: 800 (Dark)   | PIR: 1 (Motion!)   -> Relay: ON
```

## Expected Canvas Behavior
* Adjusting the simulated photoresistor to high brightness (value > 1500) keeps the relay off, even if you trigger the PIR motion sensor.
* When the photoresistor is set to dark (value < 1500) and the PIR sensor is triggered, the relay switches states and turns on.
* If the PIR sensor times out and returns to low, the relay switches back off immediately.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(LDR_PIN)` | Samples ambient light levels as an analog voltage (0 to 4095). |
| `digitalRead(PIR_PIN)` | Reads binary occupancy state from the PIR sensor. |
| `ldrValue < DARK_THRESHOLD` | Boolean check to see if the environment is dark enough. |
| `&& motionDetected == HIGH` | Logical AND operator checking if both conditions are met. |
| `digitalWrite(RELAY_PIN, HIGH)` | Drives GPIO 15 high to close the relay contacts. |

## Hardware & Safety Concept
* **Voltage Divider Principle**: LDRs change resistance depending on light exposure (low resistance in light, high resistance in dark). By wiring it in series with a fixed 10k-ohm resistor to form a voltage divider, the output voltage drops as the light level decreases, making it readable by the ADC.
* **AC Mains Safety**: Relays allow low-voltage circuits to control AC mains appliances (like a 230V lamp). Never work on open AC wiring while powered. Ensure all high-voltage connections are properly insulated within a plastic project enclosure.

## Try This! (Challenges)
1. **Off-Delay Timer**: After motion stops, keep the relay active for an additional 5 seconds (`delay(5000)`) to act as a courtesy timer so the user is not left in the dark immediately.
2. **Indicator Warning LED**: Drive a Warning LED on GPIO 15. If the system enters dark state but no motion is detected, blink the warning LED slowly to indicate the system is armed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The light turns on when it is bright | Inverted logic circuit | Check the LDR wiring or change the check logic: use `ldrValue > DARK_THRESHOLD` if using a pull-up wiring configuration. |
| Relay clicks on and off rapidly | Threshold is too close to ambient light level | Implement hysteresis or widen the threshold gap between day and night states. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [102 - Water Level Control Station](102-water-level-control-station.md)
- [103 - Smart Doorbell Alert](103-smart-doorbell-alert.md)
