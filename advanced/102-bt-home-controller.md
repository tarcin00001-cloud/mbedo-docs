# 102 - BT Home Controller

Receive single-character commands over Bluetooth from a phone app and use them to independently control a relay, an LED, and a buzzer — building a simple wireless home automation panel.

## Goal
Learn how to parse incoming Bluetooth characters with `BT.available()` / `BT.read()` and route each command through an `if/else if` chain to drive multiple independent outputs.

## What You Will Build
A phone running any Bluetooth terminal app (e.g., Serial Bluetooth Terminal) pairs with the HC-05 module. Sending:
- `'R'` toggles the relay (simulating a lamp or appliance)
- `'L'` toggles the LED indicator
- `'B'` fires the buzzer for 200 ms
- `'X'` turns everything off immediately

All state changes are echoed back over Bluetooth and printed to the Serial Monitor.

**Why these pins?** The HC-05 uses SoftwareSerial on `D10` (RX) / `D11` (TX) to avoid occupying the hardware UART. The relay control signal goes to `D7`. The LED uses `D8`. The buzzer goes to `D9`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05_bluetooth` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-05 | VCC | 5V | Power supply |
| HC-05 | GND | GND | Ground reference |
| HC-05 | TXD | D10 | SoftwareSerial RX |
| HC-05 | RXD | D11 | SoftwareSerial TX (use voltage divider on hardware) |
| Relay Module | VCC | 5V | Power supply |
| Relay Module | IN | D7 | Control signal |
| Relay Module | GND | GND | Ground reference |
| LED | Anode (+) | D8 | Through 220 ohm resistor |
| LED | Cathode (−) | GND | Ground reference |
| Passive Buzzer | + | D9 | PWM tone output |
| Passive Buzzer | − | GND | Ground reference |

## Code
```cpp
#include <SoftwareSerial.h>

SoftwareSerial BT(10, 11); // RX=D10, TX=D11

const int RELAY_PIN  = 7;
const int LED_PIN    = 8;
const int BUZZER_PIN = 9;

bool relayState = false;
bool ledState   = false;

void setup() {
  Serial.begin(9600);
  BT.begin(9600);

  pinMode(RELAY_PIN,  OUTPUT);
  pinMode(LED_PIN,    OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  digitalWrite(RELAY_PIN,  LOW);
  digitalWrite(LED_PIN,    LOW);
  digitalWrite(BUZZER_PIN, LOW);

  Serial.println("BT Home Controller Ready");
  BT.println("BT Home Controller Ready");
  BT.println("Commands: R=Relay L=LED B=Buzz X=All Off");
}

void loop() {
  if (BT.available()) {
    char cmd = BT.read();

    if (cmd == 'R' || cmd == 'r') {
      relayState = !relayState;
      digitalWrite(RELAY_PIN, relayState ? HIGH : LOW);
      Serial.print("Relay: ");
      Serial.println(relayState ? "ON" : "OFF");
      BT.print("Relay: ");
      BT.println(relayState ? "ON" : "OFF");

    } else if (cmd == 'L' || cmd == 'l') {
      ledState = !ledState;
      digitalWrite(LED_PIN, ledState ? HIGH : LOW);
      Serial.print("LED: ");
      Serial.println(ledState ? "ON" : "OFF");
      BT.print("LED: ");
      BT.println(ledState ? "ON" : "OFF");

    } else if (cmd == 'B' || cmd == 'b') {
      tone(BUZZER_PIN, 1000, 200);
      Serial.println("Buzzer: BEEP");
      BT.println("Buzzer: BEEP");

    } else if (cmd == 'X' || cmd == 'x') {
      relayState = false;
      ledState   = false;
      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(LED_PIN,   LOW);
      noTone(BUZZER_PIN);
      Serial.println("All outputs OFF");
      BT.println("All outputs OFF");

    } else {
      BT.println("Unknown command");
    }
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-05 Bluetooth Module**, **Relay Module**, **LED** with **220 ohm resistor**, and **Passive Buzzer** onto the canvas.
2. Connect HC-05: **VCC** to **5V**, **GND** to **GND**, **TXD** to **D10**, **RXD** to **D11**.
3. Connect Relay: **VCC** to **5V**, **IN** to **D7**, **GND** to **GND**.
4. Connect LED anode through 220 ohm resistor to **D8**; cathode to **GND**.
5. Connect Buzzer **+** to **D9**; **−** to **GND**.
6. Paste the code into the editor.
7. Use the default Arduino interpreted runtime.
8. Click **Run**.
9. In the MbedO BT input panel, type `R` and press Send — observe the relay toggle. Try `L`, `B`, and `X`.

## Expected Output

Terminal:
```
BT Home Controller Ready
Relay: ON
LED: ON
Buzzer: BEEP
All outputs OFF
```

## Expected Canvas Behavior
| BT Command | Relay | LED | Buzzer |
| --- | --- | --- | --- |
| `R` (first) | ON | — | — |
| `R` (second) | OFF | — | — |
| `L` (first) | — | ON | — |
| `B` | — | — | 200 ms beep |
| `X` | OFF | OFF | silent |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `SoftwareSerial BT(10, 11)` | Creates a software UART on D10/D11, leaving the hardware UART free for Serial Monitor. |
| `BT.available()` | Returns the number of bytes waiting in the Bluetooth receive buffer; non-zero means data arrived. |
| `char cmd = BT.read()` | Reads one character from the BT buffer. |
| `relayState = !relayState` | Flips the boolean state variable so each `R` command alternates between ON and OFF. |
| `tone(BUZZER_PIN, 1000, 200)` | Generates a 1 kHz tone for 200 ms on the buzzer pin. |

## Hardware & Safety Concept: SoftwareSerial and Voltage Levels
The HC-05 operates at 3.3 V logic but tolerates 5 V on its RXD input on most modules. On real hardware, protect the RXD pin with a voltage divider (e.g., 1 kΩ + 2 kΩ resistors) to drop the Arduino's 5 V TX signal to ~3.3 V.
- The TXD output from HC-05 is already 3.3 V, which is readable by the Arduino (threshold is ~2.5 V).
- SoftwareSerial is limited to lower baud rates and cannot receive and transmit simultaneously — 9600 baud is the safest choice.

## Try This! (Challenges)
1. **Status Query**: Add command `'S'` that sends back the current state of all outputs: `"Relay:ON LED:OFF"` so the phone always knows the current status.
2. **Brightness Control**: Replace the LED toggle with `analogWrite(LED_PIN, 128)` for a dim mode triggered by command `'D'`, showing three levels with `'L'`/`'D'`/`'X'`.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| No BT response | Module not paired | Pair your phone with the HC-05 (default PIN: 1234 or 0000) before opening a terminal. |
| Relay toggles twice per press | App sends `\r\n` after the character | Add `if (cmd < 32) return;` at the start of the `if` block to ignore control characters. |
| LED stays off | Polarity reversed | Swap anode and cathode; test with `digitalWrite(LED_PIN, HIGH)` in `setup()`. |
| Buzzer is silent | Wrong pin type | Ensure D9 is connected to the buzzer positive terminal, not a passive speaker with no driver. |

## Mode Notes
All patterns used here (`SoftwareSerial`, `BT.available()`, `BT.read()`, `tone()`, `digitalWrite`, `if/else if`) are fully supported by MbedO interpreted mode.

## Related Projects
- [101 - Automatic Gate](101-automatic-gate.md)
- [103 - Smart Doorbell](103-smart-doorbell.md)
- [88 - Relay Control](88-relay-control.md)
- [95 - BT LED Control](95-bt-led-control.md)
