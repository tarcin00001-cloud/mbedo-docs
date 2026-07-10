# 03 - Pico Multiple Blink

Blink two external LEDs alternately to build a railway-crossing style signal.

## Goal
Learn how to control multiple digital output pins on the Raspberry Pi Pico concurrently and coordinate opposite output states.

## What You Will Build
An alternating dual-light flasher:
- **Red LED (GP15)**: Turns ON when the Green LED is OFF.
- **Green LED (GP16)**: Turns ON when the Red LED is OFF.
- The two lights alternate state every 400 milliseconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| 220-ohm Resistors | `resistor` | Optional | Yes (two pull-downs) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Red LED | Anode (long leg) | GP15 | Control output |
| Red LED | Cathode | GND | Ground return via 220-ohm resistor |
| Green LED | Anode (long leg) | GP16 | Control output |
| Green LED | Cathode | GND | Ground return via 220-ohm resistor |

## Code
```cpp
const int RED_LED = 15;
const int GREEN_LED = 16;

void setup() {
  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);
}

void loop() {
  // Red ON, Green OFF
  digitalWrite(RED_LED, HIGH);
  digitalWrite(GREEN_LED, LOW);
  delay(400);

  // Red OFF, Green ON
  digitalWrite(RED_LED, LOW);
  digitalWrite(GREEN_LED, HIGH);
  delay(400);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Red LED**, and **Green LED** onto the canvas.
2. Connect Red LED Anode to **GP15** and Cathode to **GND**.
3. Connect Green LED Anode to **GP16** and Cathode to **GND**.
4. Paste the code into the editor, select interpreted mode, and click **Run**.
5. Watch the red and green lights flash alternately.

## Expected Output

Terminal:
```
Simulation active. Alternating GP15 (Red) and GP16 (Green).
```

## Expected Canvas Behavior
| Time (ms) | GP15 (Red LED) | GP16 (Green LED) |
| --- | --- | --- |
| 0–400 | HIGH (ON) | LOW (OFF) |
| 400–800 | LOW (OFF) | HIGH (ON) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pinMode(RED_LED, OUTPUT)` | Prepares GPIO pin 15 to output current. |
| `digitalWrite(RED_LED, HIGH)` | Drives pin 15 HIGH while pin 16 is driven LOW, lighting only the red LED. |

## Hardware & Safety Concept: Current Sharing
GPIO pins must not be connected directly together. Each LED must have its own separate current-limiting resistor tied to ground. Sharing a single resistor between two alternately driven LEDs can cause dimming or flickering due to differences in LED forward voltage drops.

## Try This! (Challenges)
1. **Three-LED Sequence**: Add a yellow LED to GP14 and configure a 3-step shifting sequence.
2. **Police Siren Sync**: Speed up the alternation to 80 ms to create an emergency vehicle flasher.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| One LED stays OFF | LED is wired backwards | Check that the longer leg (Anode) goes to the Pico pin, not GND. |

## Mode Notes
This multi-GPIO output project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [01 - Pico Blink](01-pico-blink.md)
- [04 - Pico Traffic Light](04-pico-traffic-light.md)
