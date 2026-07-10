# 90 - Pico Bluetooth LED

Control an LED wirelessly by sending text commands from a smartphone Bluetooth terminal.

## Goal
Learn how to parse character commands received over UART and use them to toggle digital outputs.

## What You Will Build
A wireless light switch:
- **HC-05 Bluetooth Module (GP0, GP1)**: Receives commands.
- **Warning LED (GP15)**: Turns ON when character `'1'` is received, and turns OFF when character `'0'` is received.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-05 | VCC | 5V | Power supply |
| HC-05 | TXD | GP1 (RX) | Data to Pico |
| HC-05 | RXD | GP0 (TX) | Data from Pico (with divider) |
| HC-05 | GND | GND | Ground reference |
| Red LED | Anode | GP15 | Lamp control |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int LED_PIN = 15;

void setup() {
  Serial1.begin(9600);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start with LED OFF
  Serial1.println("Bluetooth LED Control Active. Send 1 (ON) or 0 (OFF)");
}

void loop() {
  if (Serial1.available()) {
    char cmd = Serial1.read();

    if (cmd == '1') {
      digitalWrite(LED_PIN, HIGH);
      Serial1.println("LED: ON");
    } 
    else if (cmd == '0') {
      digitalWrite(LED_PIN, LOW);
      Serial1.println("LED: OFF");
    }
  }
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-05 Bluetooth Module**, and **Red LED** onto the canvas.
2. Connect HC-05: **TXD** to **GP1**, **RXD** to **GP0**. Connect LED to **GP15**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Type `'1'` in the Bluetooth terminal window to light up the LED, and `'0'` to turn it OFF.

## Expected Output

Terminal:
```
Bluetooth LED Control Active. Send 1 (ON) or 0 (OFF)
LED: ON
LED: OFF
```

## Expected Canvas Behavior
| Bluetooth Command | GP15 Pin state | LED Visual |
| --- | --- | --- |
| `'1'` | HIGH | ON (Red light) |
| `'0'` | LOW | OFF (Dark) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `cmd == '1'` | Compares the incoming ASCII character against the char literal `'1'` to trigger output activation. |

## Hardware & Safety Concept: ASCII Characters vs. Integers
When typing `'1'` on a terminal console, the computer transmits the **ASCII code** for the character `'1'`, which is decimal value `49`. Checking `cmd == 1` (integer) instead of `cmd == '1'` (character) is a common coding mistake, as the byte values do not match and the condition will evaluate false. Always use single quotes for character checks.

## Try This! (Challenges)
1. **Flashing Command**: Add a third command `'F'` that flashes the LED at 200 ms intervals.
2. **Dynamic Chime**: Connect a buzzer to GP14 and sound a brief confirmation tone when commands are executed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sending commands does nothing | Baud rate mismatch | Ensure your terminal emulator is configured to 9600 baud rate to match `Serial1.begin(9600)`. |

## Mode Notes
This basic serial communication project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [89 - Pico Bluetooth Serial](89-pico-bt-serial.md)
- [92 - Pico Bluetooth Relay](92-pico-bt-relay.md)
