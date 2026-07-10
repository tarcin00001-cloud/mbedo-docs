# 14 - Pico RGB Sync

Synchronize two separate RGB LEDs to flash matching primary colors in unison.

## Goal
Learn how to coordinate output states across six separate digital output pins on the Raspberry Pi Pico.

## What You Will Build
A synchronized color display:
- **RGB LED 1 (GP11, GP12, GP13)** & **RGB LED 2 (GP18, GP19, GP20)**: Both cycle through Red, Green, and Blue in perfect synchronization, changing color every 1 second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| RGB LED (CC) #1 | `led` | Yes | Yes |
| RGB LED (CC) #2 | `led` | Yes | Yes |
| 220-ohm Resistors | `resistor` | Optional | Yes (six resistors) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| RGB LED 1 | Red | GP11 | Red channel 1 |
| RGB LED 1 | Common Cathode | GND | Ground reference 1 |
| RGB LED 1 | Green | GP12 | Green channel 1 |
| RGB LED 1 | Blue | GP13 | Blue channel 1 |
| RGB LED 2 | Red | GP18 | Red channel 2 |
| RGB LED 2 | Common Cathode | GND | Ground reference 2 |
| RGB LED 2 | Green | GP19 | Green channel 2 |
| RGB LED 2 | Blue | GP20 | Blue channel 2 |

## Code
```cpp
// RGB LED 1
const int R1 = 11;
const int G1 = 12;
const int B1 = 13;

// RGB LED 2
const int R2 = 18;
const int G2 = 19;
const int B2 = 20;

void setup() {
  pinMode(R1, OUTPUT);
  pinMode(G1, OUTPUT);
  pinMode(B1, OUTPUT);
  pinMode(R2, OUTPUT);
  pinMode(G2, OUTPUT);
  pinMode(B2, OUTPUT);
}

void loop() {
  // Sync Red
  digitalWrite(R1, HIGH); digitalWrite(G1, LOW);  digitalWrite(B1, LOW);
  digitalWrite(R2, HIGH); digitalWrite(G2, LOW);  digitalWrite(B2, LOW);
  delay(1000);

  // Sync Green
  digitalWrite(R1, LOW);  digitalWrite(G1, HIGH); digitalWrite(B1, LOW);
  digitalWrite(R2, LOW);  digitalWrite(G2, HIGH); digitalWrite(B2, LOW);
  delay(1000);

  // Sync Blue
  digitalWrite(R1, LOW);  digitalWrite(G1, LOW);  digitalWrite(B1, HIGH);
  digitalWrite(R2, LOW);  digitalWrite(G2, LOW);  digitalWrite(B2, HIGH);
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **two RGB LEDs** onto the canvas.
2. Connect RGB 1: **Red** to **GP11**, **Cathode** to **GND**, **Green** to **GP12**, **Blue** to **GP13**.
3. Connect RGB 2: **Red** to **GP18**, **Cathode** to **GND**, **Green** to **GP19**, **Blue** to **GP20**.
4. Paste code, select interpreted mode, and click **Run**.
5. Watch the two RGB LEDs cycle color in unison.

## Expected Output

Terminal:
```
Simulation active. Synchronizing dual RGB LED cycles on GP11-13 and GP18-20.
```

## Expected Canvas Behavior
| Cycle Time | RGB 1 Color | RGB 2 Color |
| --- | --- | --- |
| 0–1.0 s | Red | Red |
| 1.0–2.0 s | Green | Green |
| 2.0–3.0 s | Blue | Blue |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(R1, HIGH)` | Drives GP11 and GP18 HIGH concurrently to blend the red channels on both LEDs. |

## Hardware & Safety Concept: Fanout Limits
Microcontroller pins can only deliver a maximum amount of current (typically 4mA to 12mA on the RP2040). Connecting multiple LEDs in parallel to the *same* output pin will exceed this current limit and can overheat or burn out the microcontroller pin. Using separate pins and syncing them in code is the safest method.

## Try This! (Challenges)
1. **Opposite Cycle**: Modify the code so that when LED 1 is Red, LED 2 is Green; when LED 1 is Green, LED 2 is Blue; and when LED 1 is Blue, LED 2 is Red.
2. **Police Flash**: Change colors rapidly (every 100 ms) to create an emergency response strobe setup.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Only one LED turns Green | Green pin is loose | Check the connection from GP12 (for LED 1) and GP19 (for LED 2) to the green anode pins. |

## Mode Notes
This multi-GPIO sequencer runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [07 - Pico RGB Red](07-pico-rgb-red.md)
- [10 - Pico RGB Cycle](10-pico-rgb-cycle.md)
