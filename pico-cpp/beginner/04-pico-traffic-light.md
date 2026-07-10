# 04 - Pico Traffic Light

Simulate a standard traffic light sequence using Red, Yellow, and Green LEDs.

## Goal
Learn how to sequence multiple digital output states in a specific chronological pattern.

## What You Will Build
A traffic light controller:
- **Red LED (GP13)**: ON for 3 seconds (STOP).
- **Yellow LED (GP14)**: ON for 1 second (WARNING).
- **Green LED (GP15)**: ON for 3 seconds (GO).
- The sequence loops: Green → Yellow → Red → Green.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| LED (Yellow) | `led` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| 220-ohm Resistors | `resistor` | Optional | Yes (three pull-downs) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Red LED | Anode | GP13 | Red control pin |
| Red LED | Cathode | GND | Ground return via resistor |
| Yellow LED | Anode | GP14 | Yellow control pin |
| Yellow LED | Cathode | GND | Ground return via resistor |
| Green LED | Anode | GP15 | Green control pin |
| Green LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int RED_PIN = 13;
const int YEL_PIN = 14;
const int GRN_PIN = 15;

void setup() {
  pinMode(RED_PIN, OUTPUT);
  pinMode(YEL_PIN, OUTPUT);
  pinMode(GRN_PIN, OUTPUT);
}

void loop() {
  // 1. Red Light (STOP)
  digitalWrite(RED_PIN, HIGH);
  digitalWrite(YEL_PIN, LOW);
  digitalWrite(GRN_PIN, LOW);
  delay(3000);

  // 2. Green Light (GO)
  digitalWrite(RED_PIN, LOW);
  digitalWrite(YEL_PIN, LOW);
  digitalWrite(GRN_PIN, HIGH);
  delay(3000);

  // 3. Yellow Light (WARNING)
  digitalWrite(RED_PIN, LOW);
  digitalWrite(YEL_PIN, HIGH);
  digitalWrite(GRN_PIN, LOW);
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Red LED**, **Yellow LED**, and **Green LED** onto the canvas.
2. Connect Red Anode to **GP13**, Yellow Anode to **GP14**, and Green Anode to **GP15**.
3. Connect all Cathodes to the Pico's **GND** pins.
4. Paste code, select the interpreted mode, and click **Run**.
5. Observe the standard traffic light sequence loop.

## Expected Output

Terminal:
```
Simulation active. Traffic sequence: Red (3s) -> Green (3s) -> Yellow (1s).
```

## Expected Canvas Behavior
| Time Index | Red LED (GP13) | Yellow LED (GP14) | Green LED (GP15) | Phase |
| --- | --- | --- | --- | --- |
| 0–3.0 s | HIGH (ON) | LOW (OFF) | LOW (OFF) | STOP |
| 3.0–6.0 s | LOW (OFF) | LOW (OFF) | HIGH (ON) | GO |
| 6.0–7.0 s | LOW (OFF) | HIGH (ON) | LOW (OFF) | WARNING |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(RED_PIN, HIGH)` | Drives GP13 HIGH while other pins are driven LOW to isolate the red signal. |
| `delay(3000)` | Holds the current light state for 3 seconds. |

## Hardware & Safety Concept: Pin State Reset
When microcontrollers boot, GPIO pins default to high-impedance inputs. During this boot phase, external pull-down resistors ensure that the LEDs remain completely OFF and do not flicker or activate randomly before the program controls them.

## Try This! (Challenges)
1. **Flashing Yellow**: Create a night-mode sequence where the yellow LED flashes at 500 ms intervals while Red and Green remain OFF.
2. **Pedestrian Signal**: Add a fourth blue/white LED and trigger it to turn ON only when the Red light is active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LEDs flash out of order | Pins swapped | Swap wiring connections or update pin constants in the code to match your layout. |

## Mode Notes
This multi-output sequencer runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [03 - Pico Multiple Blink](03-pico-multiple-blink.md)
- [10 - Pico Active Buzzer Patterns](06-pico-buzzer-patterns.md)
