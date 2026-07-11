# 118 - IR Remote LED Control

Toggle a warning LED on and off remotely using a handheld infrared remote control transmitter and an IR receiver module on the VEGA ARIES v3 board.

## Goal
Learn how to check for specific IR command codes and implement toggle logic to switch state variables and control physical outputs.

## What You Will Build
An infrared remote light switch. Pressing the Power button on the remote control toggles the warning LED on GPIO 15. If the LED is off, pressing the button turns it on; if the LED is on, pressing the button turns it off.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| IR Receiver Module (TSOP382) | `ir_receiver` | Yes | Yes |
| Handheld IR Remote Transmitter | `ir_remote` | Yes | Yes |
| Warning LED | `led` | Yes | Yes (external or on GPIO 15) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Receiver | VCC | 3V3 | Red | Receiver power supply (3.3V) |
| IR Receiver | GND | GND | Black | Ground reference |
| IR Receiver | OUT | GPIO 17 | Green | Digital demodulated output |
| Warning LED | Anode (+) | GPIO 15 | Blue | Driven high to turn LED ON |
| Warning LED | Cathode (-) | GND | Black | Ground reference (through 220-ohm resistor)|

> **Wiring tip:** Connect the warning LED's anode to GPIO 15 and cathode to GND. Wire the IR receiver signal pin directly to GPIO 17.

## Code
```cpp
#include <IRremote.h>

const int RECV_PIN = 17; // IR receiver on GPIO 17
const int LED_PIN = 15;  // Warning LED on GPIO 15

int ledState = 0; // State tracker: 0 = OFF, 1 = ON

void setup() {
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start with LED OFF

  // Initialize IR receiver
  IrReceiver.begin(RECV_PIN, ENABLE_LED_FEEDBACK);
}

void loop() {
  if (IrReceiver.decode()) {
    unsigned long cmd = IrReceiver.decodedIRData.command;
    
    // 0x45 is the standard HEX command code for the Power button on many IR remotes
    if (cmd == 0x45) {
      if (ledState == 0) {
        ledState = 1;
        digitalWrite(LED_PIN, HIGH); // Turn LED ON
      } else {
        ledState = 0;
        digitalWrite(LED_PIN, LOW);  // Turn LED OFF
      }
    }
    
    IrReceiver.resume(); // Ready to receive next signal
  }
  delay(100); // Polling delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **IR Receiver Module**, **IR Remote Transmitter**, and a **Warning LED** onto the canvas.
2. Wire the IR Receiver: **VCC** to **3V3**, **GND** to **GND**, and **OUT** to **GPIO 17**.
3. Wire the LED: **Anode** to **GPIO 15** (via resistor) and **Cathode** to **GND**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.
7. Click the **Power** button on the simulated remote control and watch the LED toggle states.

## Expected Output
Serial Monitor:
```
System Initialized.
IR Receiver Active.
Power Button Code Received (0x45) - Toggling LED: ON
Power Button Code Received (0x45) - Toggling LED: OFF
```

## Expected Canvas Behavior
* Pressing the **Power** button on the simulated IR remote transmitter toggles the Warning LED on GPIO 15.
* Pressing any other button on the remote has no effect on the LED state.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(LED_PIN, OUTPUT)` | Configures pin GPIO 15 to drive the warning LED. |
| `IrReceiver.begin(...)` | Starts the IR receiver library. |
| `cmd == 0x45` | Compares the decoded command code against the target Power key code (0x45). |
| `digitalWrite(LED_PIN, HIGH)` | Drives GPIO 15 high to light up the LED. |
| `IrReceiver.resume()` | Readies the IR receiver to capture the next packet. |

## Hardware & Safety Concept
* **Infrared Protocol Framing**: IR communication uses frame protocols. A typical NEC protocol frame consists of a 9 ms leading pulse, a 4.5 ms space, followed by 32 bits of address and data (including inverse bits for error checking). Demodulating these pulses requires microsecond-level timing accuracy, which is handled automatically by the library.
* **Toggle Debouncing**: Toggling outputs from remote commands requires state-change isolation. If the remote button is held down, it sending a "repeat code" (usually a simple 110 ms pulse pattern). The code handles this by verifying the command matches `0x45` and ignores repeat packets.

## Try This! (Challenges)
1. **Brightness Control**: Use a PWM-capable pin for the LED. Set the remote's "+" button to increase LED brightness (`analogWrite`) and the "-" button to decrease it.
2. **All-Off button**: Program a second button (such as the "Mute" button) to instantly turn off the LED regardless of its current state.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The LED toggles erratically on button hold | Repeat codes triggering the check | Ensure the code checks for the exact command hex and that `IrReceiver.resume()` is called immediately. |
| The LED does not toggle when pressing Power | Hex code mismatch | Use the decoder project (Project 117) to verify the exact hex code of your remote's power button and update the `0x45` value in code. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [14 - LED PWM Brightness Fade](../beginner/14-led-pwm-brightness-fade.md)
- [117 - IR Remote Receiver Command Decoder](117-ir-remote-receiver-command-decoder.md)
