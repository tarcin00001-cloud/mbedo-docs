# 05 - Traffic Light Sequence

Simulate a road traffic light that cycles through red, yellow, and green.

## Goal
Control three discrete LEDs (Red, Yellow, and Green) in a timed sequence to model a standard automotive traffic light.

## What You Will Build
A pre-packaged Traffic Light module connected to pins D9, D10, and D11 cycles through standard phases: Red (Stop) for 4 seconds -> Green (Go) for 4 seconds -> Yellow (Prepare to Stop) for 1.5 seconds -> repeating.

**Why D9, D10, and D11?** These pins provide clean digital output controls. They are kept close together to keep canvas wiring organized.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Traffic Light | `traffic_light` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes (3 required if wiring discrete LEDs) |

## Wiring
| Traffic Light Pin | Arduino Pin | Notes |
| --- | --- | --- |
| RED | D9 | Red LED control line |
| YELLOW | D10 | Yellow LED control line |
| GREEN | D11 | Green LED control line |
| GND | GND | Ground connection |

## Code
```cpp
const int RED_PIN    = 9;
const int YELLOW_PIN = 10;
const int GREEN_PIN  = 11;

void setup() {
  pinMode(RED_PIN,    OUTPUT);
  pinMode(YELLOW_PIN, OUTPUT);
  pinMode(GREEN_PIN,  OUTPUT);
  
  Serial.begin(9600);
  Serial.println("Traffic Light ready");
}

void loop() {
  // Red - stop (4 seconds)
  digitalWrite(RED_PIN, HIGH);
  digitalWrite(YELLOW_PIN, LOW);
  digitalWrite(GREEN_PIN, LOW);
  Serial.println("RED - Stop");
  delay(4000);

  // Green - go (4 seconds)
  digitalWrite(RED_PIN, LOW);
  digitalWrite(YELLOW_PIN, LOW);
  digitalWrite(GREEN_PIN, HIGH);
  Serial.println("GREEN - Go");
  delay(4000);

  // Yellow - slow down / prepare to stop (1.5 seconds)
  digitalWrite(RED_PIN, LOW);
  digitalWrite(YELLOW_PIN, HIGH);
  digitalWrite(GREEN_PIN, LOW);
  Serial.println("YELLOW - Slow");
  delay(1500);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** onto the canvas.
2. Drag **Traffic Light** onto the canvas.
3. Connect Traffic Light **RED** to Arduino **D9**.
4. Connect Traffic Light **YELLOW** to Arduino **D10**.
5. Connect Traffic Light **GREEN** to Arduino **D11**.
6. Connect Traffic Light **GND** to Arduino **GND**.
7. Paste the code into the editor.
8. Use the default Arduino interpreted runtime.
9. Click **Run**.

## Expected Output

Serial Monitor:
```
Traffic Light ready
RED - Stop
GREEN - Go
YELLOW - Slow
RED - Stop
...
```

### Expected Canvas Behavior

| Sequence Phase | Red Pin (D9) | Yellow Pin (D10) | Green Pin (D11) | Active Light | Duration |
| --- | --- | --- | --- | --- | --- |
| 1. Stop | HIGH | LOW | LOW | RED | 4.0 seconds |
| 2. Go | LOW | LOW | HIGH | GREEN | 4.0 seconds |
| 3. Warn | LOW | HIGH | LOW | YELLOW | 1.5 seconds |

The traffic light component on the canvas cycles colors in order.

## Code Walkthrough

| Line | What It Does |
| --- | --- |
| `pinMode` (setup) | Sets all three traffic light pins as outputs. |
| `digitalWrite(RED_PIN, HIGH)` | Activates the Red LED. |
| `digitalWrite(YELLOW_PIN, LOW)` | Ensures the Yellow LED is off during the red phase. |
| `delay(4000)` | Keeps the current state active for 4 seconds. |

## Hardware & Safety Concept: Discrete vs Integrated Components
A **Traffic Light Module** is a convenient integrated board. It mounts three discrete LEDs, three surface-mount resistors (often 330 ohm), and routes their grounds to a single shared pin. If you were building this using individual LEDs, you would have to wire three separate ground paths and three individual resistors manually.

## Try This! (Challenges)
1. **UK Signal Sequence**: In many countries, the sequence transitions from Red -> Red + Yellow together -> Green -> Yellow -> Red. Modify the code to implement this Red+Yellow transition stage for 1 second.
2. **Pedestrian Crossing Beep**: Add a Buzzer to D8 and generate a slow repeating beep (`tone(8, 800, 100); delay(1000);`) during the green phase to simulate a pedestrian crosswalk signal.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LEDs light up in the wrong order | Signal control wires crossed | Check the wiring table: RED to D9, YELLOW to D10, GREEN to D11. |
| All three lights stay ON together | Code logic error | Make sure each phase explicitly sets the other two unused pins to `LOW` when turning one `HIGH`. |

## Mode Notes
These patterns (`digitalWrite` sequences with `delay`) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [01 - LED Blink](01-led-blink.md)
- [04 - RGB Color Mixer](04-rgb-color-mixer.md)
