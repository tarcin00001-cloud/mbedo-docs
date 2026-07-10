# 36 - Pico PIR Motion

Detect motion events using a Passive Infrared (PIR) sensor.

## Goal
Learn how to read digital signals from motion sensor modules and trigger indicator outputs on the Raspberry Pi Pico.

## What You Will Build
An intrusion warning beacon:
- **PIR Motion Sensor (GP16)**: Reads `HIGH` when motion is detected.
- **Warning LED (GP15)**: Turns ON immediately when motion is detected, and turns OFF when the sensor goes idle.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| PIR Sensor | VCC | 5V (or 3.3V) | Power supply |
| PIR Sensor | OUT (Signal) | GP16 | Motion output signal |
| PIR Sensor | GND | GND | Ground reference |
| Red LED | Anode | GP15 | Safety warning indicator |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int PIR_PIN = 16;
const int LED_PIN = 15;

void setup() {
  pinMode(PIR_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int motionState = digitalRead(PIR_PIN);

  // PIR outputs HIGH when motion is detected
  if (motionState == HIGH) {
    digitalWrite(LED_PIN, HIGH); // Intruder! LED ON
  } else {
    digitalWrite(LED_PIN, LOW);  // Quiet, LED OFF
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **PIR Motion Sensor**, and **Red LED** onto the canvas.
2. Connect PIR: **VCC** to **5V** (or **VBUS**), **OUT** to **GP16**, **GND** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Click the PIR Sensor on canvas to trigger motion, and watch the LED light up.

## Expected Output

Terminal:
```
Simulation active. GP16 PIR guard monitor active.
```

## Expected Canvas Behavior
| PIR Sensor State | GP16 Input State | LED state (GP15) | Security status |
| --- | --- | --- | --- |
| Idle (No motion) | LOW | LOW | Clear |
| Triggered (Motion)| HIGH | HIGH | **ALERT (Motion Detected)** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `motionState == HIGH` | Verifies if the PIR output comparator has pulled the signal line to high logic voltage level. |

## Hardware & Safety Concept: Passive Infrared Sensing
PIR sensors measure changes in infrared (heat) radiation emitted by surrounding objects. Inside the sensor capsule, a pyroelectric sensor split in two halves detects the differential voltage generated when a warm body (like a human or animal) moves from one half's field of view to the other. A Fresnel lens focuses this infrared energy onto the sensor element.

## Try This! (Challenges)
1. **Acoustic Alarm**: Connect a buzzer to GP14 and sound a rapid warning beep when motion is detected.
2. **Sustained Alert**: Program the LED to stay ON for 3 seconds after the motion trigger stops.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays ON constantly | PIR sensor cooldown phase | Real PIR sensors have a hardware delay adjustment dial on the back. It may take 2–5 seconds to go back to LOW after a trigger. |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [16 - Pico Button LED](16-pico-button-led.md)
- [37 - Pico IR Obstacle](37-pico-ir-obstacle.md)
