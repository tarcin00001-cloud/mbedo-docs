# 99 - Auto Night Light

Combine an LDR and a PIR sensor so a relay-switched light turns on only when it is both dark **and** motion is detected — saving energy while providing security lighting.

## Goal
Learn how to use a logical AND condition combining two sensor readings so that two independent conditions must both be true before an output fires.

## What You Will Build
The circuit reads ambient light level from an LDR and presence from a PIR sensor every loop. If the LDR reports darkness (analog value above a threshold) **and** the PIR reports motion, the relay closes to energise a lamp and an LED indicator lights up. If either condition is false the relay opens and the LED turns off.

**Why these pins?** The LDR is wired as a voltage divider to `A0` so `analogRead` can measure light intensity. The PIR outputs a digital HIGH/LOW signal and goes to `D2`. The relay module accepts a digital control signal on `D7`. The indicator LED uses `D8` with a current-limiting resistor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| LDR (Light Dependent Resistor) | `ldr` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Yes | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes (LDR pull-down) |
| Relay Module | `relay` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| LDR | VCC leg | 5V | One leg to supply |
| LDR | OUT leg | A0 | Voltage-divider mid-point |
| 10k resistor | Between A0 and GND | — | Pull-down for LDR divider |
| PIR Sensor | VCC | 5V | Power supply |
| PIR Sensor | OUT | D2 | Digital motion signal |
| PIR Sensor | GND | GND | Ground reference |
| Relay Module | VCC | 5V | Power supply |
| Relay Module | IN | D7 | Control signal |
| Relay Module | GND | GND | Ground reference |
| LED | Anode (+) | D8 | Through 220 ohm resistor |
| LED | Cathode (−) | GND | Ground reference |

## Code
```cpp
const int LDR_PIN   = A0;
const int PIR_PIN   = 2;
const int RELAY_PIN = 7;
const int LED_PIN   = 8;

// Light threshold: readings above this value are considered "dark"
const int DARK_THRESHOLD = 600;

void setup() {
  Serial.begin(9600);
  pinMode(PIR_PIN,   INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(LED_PIN,   OUTPUT);
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(LED_PIN,   LOW);
  Serial.println("Auto Night Light Ready");
}

void loop() {
  int lightLevel = analogRead(LDR_PIN);
  int motion     = digitalRead(PIR_PIN);

  bool isDark    = (lightLevel > DARK_THRESHOLD);
  bool hasMotion = (motion == HIGH);

  Serial.print("Light: ");
  Serial.print(lightLevel);
  Serial.print(" | Motion: ");
  Serial.print(hasMotion ? "YES" : "NO");
  Serial.print(" | Relay: ");

  if (isDark && hasMotion) {
    digitalWrite(RELAY_PIN, HIGH);
    digitalWrite(LED_PIN,   HIGH);
    Serial.println("ON");
  } else {
    digitalWrite(RELAY_PIN, LOW);
    digitalWrite(LED_PIN,   LOW);
    Serial.println("OFF");
  }

  delay(500);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **LDR**, **PIR Motion Sensor**, **LED**, **220 ohm resistor**, and **Relay Module** onto the canvas.
2. Connect LDR: one leg to **5V**, the other to **A0**; place a 10 kΩ resistor between **A0** and **GND**.
3. Connect PIR: **VCC** to **5V**, **OUT** to **D2**, **GND** to **GND**.
4. Connect Relay: **VCC** to **5V**, **IN** to **D7**, **GND** to **GND**.
5. Connect LED anode through the 220 ohm resistor to **D8**; LED cathode to **GND**.
6. Paste the code into the editor.
7. Use the default Arduino interpreted runtime.
8. Click **Run**.
9. Double-click the LDR slider and drag it to a high value (dark) and double-click the PIR to toggle motion — observe the relay and LED.

## Expected Output

Terminal:
```
Auto Night Light Ready
Light: 820 | Motion: YES | Relay: ON
Light: 820 | Motion: NO  | Relay: OFF
Light: 200 | Motion: YES | Relay: OFF
```

## Expected Canvas Behavior
| LDR Value | PIR State | Relay | LED |
| --- | --- | --- | --- |
| 820 (dark) | HIGH (motion) | ON | ON |
| 820 (dark) | LOW (no motion) | OFF | OFF |
| 200 (bright) | HIGH (motion) | OFF | OFF |
| 200 (bright) | LOW (no motion) | OFF | OFF |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `lightLevel > DARK_THRESHOLD` | Returns `true` only when ambient light is low (high resistance → high voltage at A0). |
| `motion == HIGH` | Returns `true` when the PIR detects infrared heat movement in its field of view. |
| `isDark && hasMotion` | Logical AND — both conditions must be simultaneously true before the relay energises. |
| `digitalWrite(RELAY_PIN, HIGH)` | Closes the relay coil, connecting the load circuit (lamp) to mains or a power supply. |

## Hardware & Safety Concept: Relay-Switched Loads
A relay is an electromechanical switch — a small control current through the coil creates a magnetic field that physically closes (or opens) a separate set of contacts rated for much higher currents and voltages.
- The Arduino 5 V signal drives the coil side; the load side is electrically isolated.
- Always use a flyback diode across the relay coil on real hardware to suppress the voltage spike when the coil de-energises.
- Most relay modules include the diode and a transistor driver on-board, making them safe to connect directly to an Arduino output pin.

## Try This! (Challenges)
1. **Adjustable Threshold**: Wire a potentiometer to `A1` and use `map(analogRead(A1), 0, 1023, 300, 900)` to set `DARK_THRESHOLD` dynamically without re-uploading code.
2. **Hold Timer**: Keep the relay on for a fixed number of loop iterations after the last motion event, rather than switching off immediately, by counting loops with an integer counter.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay is always ON in daylight | LDR wired incorrectly | Confirm the 10 kΩ pull-down is between A0 and GND, not between A0 and 5V. |
| PIR never triggers | PIR warm-up delay | Allow 30–60 seconds after power-on for the PIR pyroelectric sensor to stabilise. |
| Relay clicks but LED stays off | LED polarity reversed | Swap anode and cathode connections. |
| Serial always prints `OFF` | Threshold too high | Lower `DARK_THRESHOLD` or cover the LDR with your hand to test dark condition. |

## Mode Notes
All patterns used here (`analogRead`, `digitalRead`, `digitalWrite`, `if/else`, `Serial.print`) are fully supported by MbedO interpreted mode.

## Related Projects
- [55 - LDR Night Light](../intermediate/55-ldr-night-light.md)
- [57 - PIR Motion Alert](../intermediate/57-pir-motion-alert.md)
- [88 - Relay Control](88-relay-control.md)
- [100 - Smart Fan](100-smart-fan.md)
