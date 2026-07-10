# 21 - Pico Touch Lamp

Build a touch-activated lamp utilizing a TTP223 capacitive touch sensor and an LED.

## Goal
Learn how to interface capacitive touch sensors with digital GPIO input pins on the Raspberry Pi Pico.

## What You Will Build
A touch-sensitive lamp:
- **TTP223 Touch Sensor (GP16)**: Monitored via digital input. Reads `HIGH` when touched.
- **Warning LED (GP15)**: Illuminates immediately when the touch pad is touched, and turns OFF when released.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| TTP223 Touch Sensor | `ttp223` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Touch Sensor | VCC | 3.3V | Power supply |
| Touch Sensor | I/O (SIG) | GP16 | Digital touch output |
| Touch Sensor | GND | GND | Ground reference |
| Red LED | Anode | GP15 | Lamp control |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int TOUCH_PIN = 16;
const int LED_PIN   = 15;

void setup() {
  pinMode(TOUCH_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int touchState = digitalRead(TOUCH_PIN);

  // TTP223 outputs HIGH when touched, LOW when idle
  if (touchState == HIGH) {
    digitalWrite(LED_PIN, HIGH); // Lamp ON
  } else {
    digitalWrite(LED_PIN, LOW);  // Lamp OFF
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **TTP223 Touch Sensor**, and **Red LED** onto the canvas.
2. Connect Touch Sensor: **VCC** to **3.3V** (or **3V3**), **I/O** (or **SIG**) to **GP16**, **GND** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Click and hold the Touch Sensor to turn on the lamp.

## Expected Output

Terminal:
```
Simulation active. Reading TTP223 on GP16.
```

## Expected Canvas Behavior
| Touch Sensor (GP16) | LED state (GP15) | Lamp Visual |
| --- | --- | --- |
| LOW (Idle) | LOW | OFF (Dark) |
| HIGH (Touched) | HIGH | ON (Red light) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `touchState == HIGH` | Detects the capacitive activation state reported by the TTP223 sensor chip. |

## Hardware & Safety Concept: Capacitive Touch Sensing
Capacitive touch sensors like the TTP223 measure the change in electrical capacitance of the sensor pad. When a human finger approaches, the body's natural capacitance alters the charge level on the copper pad. The internal IC detects this minor shift, filters noise, and outputs a clean digital `HIGH` logic signal.

## Try This! (Challenges)
1. **Touch Alarm**: Add an active buzzer on GP14 that sounds only when the pad is touched.
2. **Delayed Off**: Program the LED to remain ON for 2 seconds after the finger is removed from the touchpad.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Touch sensor does not register | Missing VCC connection | Ensure the touch module VCC pin is wired to the Pico's 3.3V pin. |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [16 - Pico Button LED](16-pico-button-led.md)
- [22 - Pico Touch Toggle](22-pico-touch-toggle.md)
