# 12 - Dual RGB LED Sync

Synchronize the color states of the onboard RGB LED and an external RGB LED connected to the VEGA ARIES v3 board.

## Goal
Learn how to coordinate multiple sets of digital output pins to synchronize state changes across internal and external indicators.

## What You Will Build
The onboard RGB LED and an external common-cathode RGB LED (connected to pins `GPIO 15`, `GPIO 13`, and `GPIO 12`) are synchronized to cycle through Red, Green, and Blue together every 1000 milliseconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| External RGB LED (Common Cathode) | `rgbled` | Yes | Yes |
| 3x 220 Ω Resistors | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| External RGB | Red Anode | GPIO 15 | Red | Controls external Red channel (via resistor) |
| External RGB | Green Anode | GPIO 13 | Green | Controls external Green channel (via resistor) |
| External RGB | Blue Anode | GPIO 12 | Blue | Controls external Blue channel (via resistor) |
| External RGB | Common Cathode | GND | Black | Common ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with each of the three positive anodes of the external RGB LED to balance currents.

## Code
```cpp
// Dual RGB LED Sync - VEGA ARIES v3
const int EXT_RED = 15;
const int EXT_GREEN = 13;
const int EXT_BLUE = 12;

void setup() {
  // Configure onboard RGB pins as outputs
  pinMode(LED_R, OUTPUT);
  pinMode(LED_G, OUTPUT);
  pinMode(LED_B, OUTPUT);
  
  // Configure external RGB pins as outputs
  pinMode(EXT_RED, OUTPUT);
  pinMode(EXT_GREEN, OUTPUT);
  pinMode(EXT_BLUE, OUTPUT);
}

void loop() {
  // 1. Synchronized RED
  digitalWrite(LED_R, HIGH);  digitalWrite(LED_G, LOW);   digitalWrite(LED_B, LOW);
  digitalWrite(EXT_RED, HIGH); digitalWrite(EXT_GREEN, LOW);  digitalWrite(EXT_BLUE, LOW);
  delay(1000);
  
  // 2. Synchronized GREEN
  digitalWrite(LED_R, LOW);   digitalWrite(LED_G, HIGH);  digitalWrite(LED_B, LOW);
  digitalWrite(EXT_RED, LOW);  digitalWrite(EXT_GREEN, HIGH); digitalWrite(EXT_BLUE, LOW);
  delay(1000);
  
  // 3. Synchronized BLUE
  digitalWrite(LED_R, LOW);   digitalWrite(LED_G, LOW);   digitalWrite(LED_B, HIGH);
  digitalWrite(EXT_RED, LOW);  digitalWrite(EXT_GREEN, LOW);  digitalWrite(EXT_BLUE, HIGH);
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, an **RGB LED**, and **3 Resistors** onto the canvas.
2. Wire the RGB anodes (Red, Green, Blue) to **GPIO 15, 13, and 12** respectively through resistors.
3. Wire the common pin (GND/Cathode) to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Observe the onboard and external RGB LED widgets switching colors together.

## Expected Output
Serial Monitor:
```
System Initialized.
Sync State: RED
Sync State: GREEN
Sync State: BLUE
```

## Expected Canvas Behavior
* Both the onboard RGB LED and the external RGB LED widget change colors synchronously from Red to Green to Blue in 1-second cycles.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(EXT_RED, OUTPUT)` | Prepares the external Red channel pin. |
| `digitalWrite(LED_R, HIGH)` | Activates the onboard Red channel. |
| `digitalWrite(EXT_RED, HIGH)` | Activates the external Red channel, creating the synchronized color effect. |

## Hardware & Safety Concept: Common Anode vs. Common Cathode LEDs
* **Common Anode vs. Common Cathode**: RGB LEDs share a common connection pin. Common Cathode modules link all negative terminals to ground, requiring a HIGH state to light a channel. Common Anode modules connect to VCC, requiring a LOW state to sink current and light a channel.
* **Pin Current Limits**: Sourcing current for six LED channels simultaneously (especially if mixing white colors) can load the SoC regulator. Keep resistors at 220 Ω or higher to limit the pin loads to under 8 mA each.

## Try This! (Challenges)
1. **Alternating Sync**: Change the logic so when the onboard LED is Red, the external LED is Green, and they swap.
2. **Cyan-Magenta-Yellow Cycle**: Synchronize the LEDs to cycle through C-M-Y mixed colors instead of primary colors.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| One of the external channels does not work | Wiring polarity error | Ensure you are using a Common Cathode RGB LED. If you use a Common Anode module, the logic states will appear inverted (LOW turns it ON, HIGH turns it OFF) |
| Colors are mismatched between onboard/external | Wires crossed | Double-check that Red goes to GPIO 15, Green to 13, and Blue to 12. Swapped pins will cause color mismatch |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [11 - 5V DC Fan switch (via Relay/Transistor)](11-5v-dc-fan-switch.md)
- [13 - Buzzer warning pulse](13-buzzer-warning-pulse.md) (Next project)
