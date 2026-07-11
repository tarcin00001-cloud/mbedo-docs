# 117 - IR Remote Receiver Command Decoder

Capture and decode infrared (IR) command signals from a remote control transmitter using an IR receiver module on the VEGA ARIES v3 board.

## Goal
Learn how to interface infrared receivers, decode remote protocol structures (such as NEC or Sony), and print decoded command hex codes to the Serial Monitor without loops.

## What You Will Build
An infrared terminal logger. Pressing buttons on a standard handheld IR remote transmitter sends encoded infrared light pulses. The IR receiver decodes these signals, and the board prints the corresponding protocol name and raw hexadecimal command value to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| IR Receiver Module (TSOP382) | `ir_receiver` | Yes | Yes |
| Handheld IR Remote Transmitter | `ir_remote` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Receiver | VCC | 3V3 | Red | Receiver power supply (3.3V) |
| IR Receiver | GND | GND | Black | Ground reference |
| IR Receiver | OUT | GPIO 17 | Green | Digital demodulated output |

> **Wiring tip:** Demodulating IR receivers (like the TSOP382 series) have three pins: VCC, GND, and OUT. Out connects directly to ARIES GPIO 17. Keep power lines clean to avoid false decode errors.

## Code
```cpp
#include <IRremote.h>

const int RECV_PIN = 17; // IR receiver on GPIO 17

void setup() {
  Serial.begin(9600);
  
  // Initialize the IR receiver and enable active status LED feedback
  IrReceiver.begin(RECV_PIN, ENABLE_LED_FEEDBACK);
  Serial.println("IR Receiver Decoder Ready");
}

void loop() {
  // Check if a packet of IR data has been demodulated and received
  if (IrReceiver.decode()) {
    Serial.print("Protocol: ");
    Serial.print(IrReceiver.decodedIRData.protocol);
    
    Serial.print(" | Command Hex: 0x");
    Serial.println(IrReceiver.decodedIRData.command, HEX);
    
    // Prepare receiver to capture the next packet
    IrReceiver.resume(); 
  }
  delay(100); // Polling delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **IR Receiver Module**, and **IR Remote Transmitter** components onto the canvas.
2. Wire the IR Receiver: **VCC** to **3V3**, **GND** to **GND**, and **OUT** to **GPIO 17**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. Open the Serial Monitor. Click buttons on the simulated remote control and view the decoded logs.

## Expected Output
Serial Monitor:
```
System Initialized.
IR Receiver Decoder Ready
Protocol: 1 | Command Hex: 0x30
Protocol: 1 | Command Hex: 0x18
```

## Expected Canvas Behavior
* Pressing the power button or digit buttons on the simulated IR remote transmitter flashes the simulated IR receiver status indicator.
* The Serial Monitor prints the decoded protocol ID and corresponding hex command codes (e.g. `Command Hex: 0x30` for button 1).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <IRremote.h>` | Imports the standard IRremote library for parsing modulated pulses. |
| `IrReceiver.begin(...)` | Sets up the timer interrupts and input capture pin on GP17. |
| `IrReceiver.decode()` | Non-blocking function returning true when a valid IR frame is demodulated. |
| `decodedIRData.command` | Variable holding the parsed digital byte representing the key code. |
| `IrReceiver.resume()` | Resets internal buffers to prepare for the next infrared pulse burst. |

## Hardware & Safety Concept
* **Demodulation at 38kHz**: IR remotes send data by pulsing an infrared LED at 38kHz (or 40kHz) to distinguish signals from ambient light (sunlight or light bulbs). The TSOP382 receiver contains an internal optical bandpass filter, amplifier, and demodulator. It outputs a clean digital `LOW` pulse when it detects 38kHz IR light, and `HIGH` when idle.
* **Optical Line of Sight**: Infrared is a light-based communication medium. It requires line-of-sight pathing. Solid physical obstacles will block the signal, though reflective surfaces (like light-colored walls) can bounce signals to the receiver.

## Try This! (Challenges)
1. **Specific Key Action**: Read key hex values. If the command code matches the remote's "Volume Up" button, light up the onboard Green LED (`LED_G` on GPIO 24) for 500 ms.
2. **Signal Blink**: Pulse a warning LED on GPIO 15 for 50 ms every time a valid IR frame is successfully decoded.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Received codes show as `0x0` or wrong protocols | Library version mismatch or code lag | Ensure no long delay blocks exist in the loop. The receiver needs to poll code signals promptly. |
| No output displays when pressing remote | Wrong wiring connections | Confirm the OUT pin of the receiver is wired to ARIES GPIO 17, not to an analog pin. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [101 - Automatic Barrier Gate](101-automatic-barrier-gate.md)
- [118 - IR Remote LED Control](118-ir-remote-led-control.md)
