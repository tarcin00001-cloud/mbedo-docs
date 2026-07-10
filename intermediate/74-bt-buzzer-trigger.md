# 74 - BT Buzzer Trigger

Trigger a warning beep on a buzzer remotely by sending a Bluetooth command.

## Goal
Learn how to parse serial commands and trigger timed frequencies (`tone()`) on a buzzer actuator.

## What You Will Build
The Arduino polls the Bluetooth stream. When a user sends the character `'B'` (Beep) over Bluetooth, the buzzer connected to pin D8 sounds a 1000 Hz alert beep for 200 ms.

**Why D2, D3, and D8?** Pins D2/D3 handle the Bluetooth communication. Pin D8 drives the warning buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05_bluetooth` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| 100 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-05 Module | TXD | D2 | Software RX |
| HC-05 Module | RXD | D3 | Software TX (Use divider on hardware) |
| HC-05 Module | VCC | 5V | Power supply |
| HC-05 Module | GND | GND | Ground reference |
| Buzzer | Pin 1 (Positive) | D8 | Audio control line |
| Buzzer | Pin 2 (Negative) | GND | Ground reference |

## Code
```cpp
#include <SoftwareSerial.h>

SoftwareSerial BT(2, 3); // RX = D2, TX = D3

const int BUZZ_PIN = 8;

void setup() {
  pinMode(BUZZ_PIN, OUTPUT);
  
  Serial.begin(9600);
  BT.begin(9600);
  
  Serial.println("Bluetooth Buzzer Trigger Ready");
}

void loop() {
  if (BT.available() > 0) {
    char command = BT.read();
    
    Serial.print("Received: ");
    Serial.println(command);
    
    // Check if character is 'B'
    if (command == 'B' || command == 'b') {
      Serial.println("[ACTION] Triggering Beep!");
      
      // Sound a 1000 Hz beep for 200 milliseconds
      tone(BUZZ_PIN, 1000);
      delay(200);
      noTone(BUZZ_PIN);
      
      BT.println("Buzzer beeped.");
    }
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-05 Bluetooth Module**, and **Buzzer** onto the canvas.
2. Connect HC-05: **TXD** to **D2**, **RXD** to **D3**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect Buzzer **pin 1** to Arduino **D8** and **pin 2** to Arduino **GND**.
4. Paste the code into the editor.
5. Select the **Compiled Arduino Uno** runtime.
6. Click **Build and Run** (or **Run**).
7. Open the **Terminal** tab in the bottom dock.
8. Send character `'B'` over the simulated Bluetooth port to trigger the beep.

## Expected Output

Terminal:
```
Bluetooth Buzzer Trigger Ready
Received: B
[ACTION] Triggering Beep!
...
```
The buzzer vibrates and sounds on the canvas for 200 ms when the command is received.

## Expected Canvas Behavior

| Sent Bluetooth Command | Pin D8 Output | Buzzer state | Status Return over BT |
| --- | --- | --- | --- |
| 'B' | 1000 Hz square wave | Beeping (200 ms) | "Buzzer beeped." |
| Other | LOW (0V) | Silent | None |

The buzzer triggers instantly when the command character matches.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `if (command == 'B' || command == 'b')` | Logical OR comparison, accepting both uppercase and lowercase command characters. |
| `tone(BUZZ_PIN, 1000)` | Generates a 1000 Hz square wave signal to drive the piezo buzzer element. |

## Hardware & Safety Concept: Active vs Passive Buzzers
Buzzers are available in two main configurations:
- **Active Buzzers**: Contain internal oscillator circuits. Simply applying a constant DC voltage (e.g. `digitalWrite(HIGH)`) causes them to sound at a fixed frequency.
- **Passive Buzzers**: Do not contain oscillators. They act like tiny speakers and require an AC frequency signal (using the `tone()` function) to generate sound, allowing you to play different notes and melodies.

## Try This! (Challenges)
1. **Multi-Tone Alert**: Add a new command `'S'` that triggers a wailing siren pattern (alternate high and low tones).
2. **Frequency Dial**: Change the code so that sending numbers `'1'` to `'5'` plays different note pitches (e.g. C5, D5, E5, F5, G5).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not beep | Incorrect runtime selected | Verify that you have selected the **Compiled Arduino Uno** runtime. |
| Click sound instead of tone | Ground wire missing | Ensure Buzzer Pin 2 connects directly to GND. |

## Mode Notes
This project **requires the Compiled Arduino Uno** runtime because custom byte parsing logic is not processed by the interpreted mode runner.

## Related Projects
- [09 - Doorbell Tone](../beginner/09-doorbell-tone.md)
- [73 - BT LED Control](73-bt-led-control.md)
- [75 - BT Relay Switch](75-bt-relay-switch.md)
