# 108 - Intrusion Detector Alarm

Build a security barrier system using a laser transmitter and a photoresistor (LDR) to detect when a beam is broken, triggering an alarm relay on the VEGA ARIES v3 board.

## Goal
Learn how to align active light transmitters with analog sensors, set precision tripwire thresholds, and operate relays for security alerts.

## What You Will Build
A laser tripwire security system. The board turns ON a laser module that aims directly at a photoresistor. If an intruder breaks the laser path, the light level sensed by the LDR drops instantly, triggering a relay to power an external security alarm or light.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Laser Transmitter Module | `laser` | Yes | Yes |
| Photoresistor (LDR) | `ldr` | Yes | Yes |
| 5V Relay Module (Siren) | `relay` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Laser Module | VCC | GPIO 15 | Red | Laser power control pin |
| Laser Module | GND | GND | Black | Ground reference |
| LDR Circuit | VCC | 3V3 | Red | Voltage divider supply (3.3V) |
| LDR Circuit | OUT (Signal) | ADC1 (GP27) | White | Analog light level output |
| LDR Circuit | GND | GND | Black | Ground reference |
| Relay Module | VCC | 5V | Red | Relay coil operating voltage |
| Relay Module | GND | GND | Black | Ground reference |
| Relay Module | IN | GPIO 13 | Blue | Control signal pin |

> **Wiring tip:** Connect the laser transmitter's VCC pin directly to ARIES GPIO 15. This allows the program to turn the laser ON at setup and monitor the photoresistor's response.

## Code
```cpp
const int LASER_PIN = 15;   // Laser Transmitter on GPIO 15
const int LDR_PIN = 27;     // Photoresistor (LDR) on ADC1 (GP27)
const int RELAY_PIN = 13;   // Siren/Alarm Relay on GPIO 13

// Threshold value for tripwire (under direct laser light, LDR reads high)
// A value below 1800 indicates the laser beam is broken.
const int TRIGGER_THRESHOLD = 1800; 

void setup() {
  pinMode(LASER_PIN, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);

  // Power the laser transmitter ON
  digitalWrite(LASER_PIN, HIGH);
  
  // Start with alarm relay OFF
  digitalWrite(RELAY_PIN, LOW);
}

void loop() {
  int lightLevel = analogRead(LDR_PIN);

  // If the light level drops below threshold, the beam is broken
  if (lightLevel < TRIGGER_THRESHOLD) {
    digitalWrite(RELAY_PIN, HIGH); // Sound Siren
  } else {
    digitalWrite(RELAY_PIN, LOW);  // Turn off Siren
  }

  delay(50); // Fast scan rate to detect quick movements
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Laser Transmitter Module**, **Photoresistor (LDR)**, and **5V Relay Module** components onto the canvas.
2. Wire the Laser: **VCC** to **GPIO 15** and **GND** to **GND**.
3. Wire the LDR Circuit: **VCC** to **3V3**, **OUT** to **ADC1 (GP27)**, and **GND** to **GND**.
4. Wire the Relay: **VCC** to **5V**, **GND** to **GND**, and **IN** to **GPIO 13**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Laser Enabled. Monitoring beam...
Beam status: OK (ADC: 3500)
...
Beam status: BROKEN (ADC: 800) - INTRUDER ALERT!
```

## Expected Canvas Behavior
* The simulated laser turns ON (visualized by a beam or state change).
* Changing the photoresistor's simulated brightness level to high (representing the laser shining on it) keeps the relay off.
* Dragging the photoresistor slider down to low brightness (representing a broken laser beam) triggers the relay to switch states immediately.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(LASER_PIN, OUTPUT)` | Configures pin GPIO 15 to drive the laser transmitter module. |
| `digitalWrite(LASER_PIN, HIGH)` | Applies 3.3V to power up the laser emitter. |
| `analogRead(LDR_PIN)` | Samples the LDR voltage divider to check if the laser is shining on it. |
| `lightLevel < TRIGGER_THRESHOLD` | Detects when the light level drops, indicating a beam break. |
| `digitalWrite(RELAY_PIN, HIGH)` | Drives GPIO 13 high to engage the alarm. |

## Hardware & Safety Concept
* **Laser Safety**: Use low-power Class 1 or Class 2 laser modules (under 1mW) for hobby projects. Avoid looking directly into the laser beam or pointing it at reflective surfaces towards people's eyes, as it can cause permanent retinal damage.
* **Laser Alignment & Ambient Light Shielding**: In physical installations, fit a small black tube or sleeve over the LDR. This blocks stray ambient sunlight or indoor lighting from reaching the sensor, ensuring that only the direct, concentrated laser beam is measured.

## Try This! (Challenges)
1. **Latching Security Alarm**: Modify the code so that once the laser beam is broken, the relay remains ON (latched) indefinitely until a reset button on GPIO 16 is pressed.
2. **Intruder Counter**: Track the number of times the laser beam has been broken. Print the count on the Serial Monitor every time a new break event occurs.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly even without blocker | Laser misaligned or ambient light too low | Make sure the physical laser is centered on the LDR. Check if the threshold in the code is set too high. |
| Laser does not light up | Connected to wrong pin | Confirm the laser module is connected to GPIO 15 and that the pin is set to HIGH. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [104 - Night Light Controller](104-night-light-controller.md)
- [106 - Shake Alarm System](106-shake-alarm-system.md)
