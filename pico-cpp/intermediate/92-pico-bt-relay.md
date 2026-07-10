# 92 - Pico Bluetooth Relay

Actuate high-power relay switches wirelessly by sending text commands from a smartphone Bluetooth terminal.

## Goal
Learn how to parse serial character commands and switch high-power load relays wirelessly.

## What You Will Build
A wireless appliance switch:
- **HC-05 Bluetooth Module (GP0, GP1)**: Receives commands.
- **Relay Module (GP10)**: Activates (closes Normally Open contact) when command `'1'` is received, and deactivates when `'0'` is received.
- **Warning LED (GP15)**: Toggles in sync with the relay state.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| Relay Module | `relay` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-05 | VCC | 5V | Power supply |
| HC-05 | TXD | GP1 (RX) | Data to Pico |
| HC-05 | RXD | GP0 (TX) | Data from Pico (with divider) |
| HC-05 | GND | GND | Ground reference |
| Relay Module | VCC | 5V | Coil power |
| Relay Module | IN | GP10 | Relay signal |
| Relay Module | GND | GND | Ground return |
| Red LED | Anode | GP15 | Status indicator |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int RELAY_PIN = 10;
const int RED_LED   = 15;

void setup() {
  Serial1.begin(9600);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(RED_LED, LOW);
  Serial1.println("Bluetooth Relay Switch Active. Send 1 (ON) or 0 (OFF)");
}

void loop() {
  if (Serial1.available()) {
    char cmd = Serial1.read();

    if (cmd == '1') {
      digitalWrite(RELAY_PIN, HIGH);
      digitalWrite(RED_LED, HIGH);
      Serial1.println("Relay: ACTIVE");
    } 
    else if (cmd == '0') {
      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(RED_LED, LOW);
      Serial1.println("Relay: INACTIVE");
    }
  }
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-05 Bluetooth Module**, **Relay Module**, and **Red LED** onto the canvas.
2. Connect HC-05: **TXD** to **GP1**, **RXD** to **GP0**. Connect Relay to **GP10**, LED to **GP15**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Type `'1'` (Active) or `'0'` (Inactive) in the Bluetooth terminal window to toggle the relay.

## Expected Output

Terminal:
```
Bluetooth Relay Switch Active. Send 1 (ON) or 0 (OFF)
Relay: ACTIVE
Relay: INACTIVE
```

## Expected Canvas Behavior
| Bluetooth Command | Relay (GP10) state | LED (GP15) state |
| --- | --- | --- |
| `'1'` | HIGH (Closed) | HIGH (ON) |
| `'0'` | LOW (Open) | LOW (OFF) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(RELAY_PIN, HIGH)` | Drives GP10 HIGH to actuate the relay contact, switching the connected appliance ON. |

## Hardware & Safety Concept: Industrial Noise Suppression
Relays switching inductive AC loads (like fans, motors, or fluorescent tubes) can generate electrical noise and sparks across the contact points. This high-frequency noise can travel back through the air or power rails, causing the microcontroller to reset or freeze. Placing an **RC Snubber circuit** (a resistor and capacitor in series) in parallel with the relay contacts absorbs this energy, suppressing sparks and protecting the controller.

## Try This! (Challenges)
1. **Pulse Output**: Add a command `'P'` that turns the relay ON for 1.5 seconds and then turns it OFF automatically.
2. **Audio Feedback**: Connect a buzzer on GP14 and sound a brief tone on command execution.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay does not switch on command | Missing power on VCC | Check that the relay module VCC pin is connected to the Pico 5V pin (VBUS), as 3.3V may not be enough to actuate the relay coil. |

## Mode Notes
This basic serial communication project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [11 - Pico Relay Toggle](../../beginner/11-pico-relay-toggle.md)
- [90 - Pico Bluetooth LED](90-pico-bt-led.md)
