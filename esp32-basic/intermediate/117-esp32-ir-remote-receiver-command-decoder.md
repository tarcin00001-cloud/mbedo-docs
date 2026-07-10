# 117 - ESP32 IR Remote Receiver Command Decoder

Decode infrared (IR) signals from a handheld remote control and print their unique button codes to the Serial Monitor.

## Goal
Learn how to interface IR receiver modules using the `IRremote` library, decode infrared pulse-width modulated protocols (NEC, Sony, RC5), and map raw hex values to buttons.

## What You Will Build
An IR receiver module is connected to GPIO 15. The ESP32 listens for incoming infrared light pulse bursts from a remote control. When a button is pressed, the ESP32 decodes the protocol type, extracts the unique hexadecimal value, and prints it to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| IR Receiver Module (VS1838B) | `ir_receiver` | Yes | Yes |
| IR Handheld Remote Control | `ir_remote` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Receiver | OUT (Signal) | GPIO15 | Yellow | Demodulated pulse stream |
| IR Receiver | VCC | 3V3 | Red | Power |
| IR Receiver | GND | GND | Black | Ground reference |

> **Wiring tip:** The VS1838B receiver runs on 3.3V. Connect the OUT pin directly to GPIO 15. Keep it clear of strong fluorescent lights to prevent infrared noise.

## Code
```cpp
// IR Remote Receiver Command Decoder
#include <IRremote.h>

const int RECV_PIN = 15;

void setup() {
  Serial.begin(115200);
  Serial.println("Initialising IR Receiver...");
  
  // Start the receiver
  IrReceiver.begin(RECV_PIN, ENABLE_LED_FEEDBACK); 
  
  Serial.println("IR Receiver ready. Press any remote button.");
}

void loop() {
  // Check if an IR signal has been received and successfully decoded
  if (IrReceiver.decode()) {
    
    // Print raw data in hexadecimal format
    Serial.print("Protocol: ");
    Serial.print(getProtocolString(IrReceiver.decodedIRData.protocol));
    Serial.print(" | Address: 0x");
    Serial.print(IrReceiver.decodedIRData.address, HEX);
    Serial.print(" | Command (Hex): 0x");
    Serial.print(IrReceiver.decodedIRData.command, HEX);
    Serial.println();
    
    // Translate command codes to button labels
    // (These hex codes correspond to standard NEC/Car MP3 remotes)
    switch (IrReceiver.decodedIRData.command) {
      case 0x45: Serial.println("Button: POWER"); break;
      case 0x46: Serial.println("Button: MODE"); break;
      case 0x47: Serial.println("Button: MUTE"); break;
      case 0x44: Serial.println("Button: PLAY/PAUSE"); break;
      case 0x40: Serial.println("Button: VOL+"); break;
      case 0x19: Serial.println("Button: VOL-"); break;
      case 0xC:  Serial.println("Button: 1"); break;
      case 0x18: Serial.println("Button: 2"); break;
      case 0x5E: Serial.println("Button: 3"); break;
      default:   Serial.println("Button: Unknown Command"); break;
    }
    
    Serial.println("----------------------------------------");
    
    // Resume receiver to listen for next signal
    IrReceiver.resume(); 
  }
  delay(100); // Poll rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **IR Receiver**, and **IR Remote** onto the canvas.
2. Wire the IR Receiver OUT to **GPIO15**.
3. Paste the code and click **Run**.
4. Click buttons on the virtual IR Remote widget. Watch the decoded button names and hex codes print to the Serial Monitor.

## Expected Output
Serial Monitor:
```
Initialising IR Receiver...
IR Receiver ready. Press any remote button.
Protocol: NEC | Address: 0x0 | Command (Hex): 0x45
Button: POWER
----------------------------------------
Protocol: NEC | Address: 0x0 | Command (Hex): 0xC
Button: 1
```

## Expected Canvas Behavior
* Clicking any key on the remote widget flashes the IR receiver widget indicator.
* The Serial Monitor outputs the protocol name, hex command value, and translated button labels instantly.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `IrReceiver.begin(RECV_PIN, ...)` | Configures hardware timers to sample pulses on GPIO 15 and enables the onboard LED to flash on scans. |
| `IrReceiver.decode()` | Non-blocking poll returns true if a valid pulse frame was decoded. |
| `IrReceiver.decodedIRData.command` | Retrieves the decoded 8-bit or 16-bit command value corresponding to the pressed button. |
| `IrReceiver.resume()` | Clears buffers and prepares the internal receiver interrupt routine for the next packet. |

## Hardware & Safety Concept: Infrared Remote Protocols
IR remotes transmit data by flashing an infrared LED at a specific frequency (usually **38 kHz**) to avoid interference from ambient sunlight. When you press a button, the remote sends a start pulse followed by binary data where 0s and 1s are encoded as different pulse width durations (NEC protocol: logical 1 is a 560 µs pulse followed by 1.69 ms space). The VS1838B receiver contains an optical filter and demodulator chip that strips out the 38 kHz carrier frequency, outputting a clean logic-level pulse train to the ESP32.

## Try This! (Challenges)
1. **Repeat Command Detection**: Detect if a button is held down (which outputs repeat codes `0xFFFFFFFF` in some protocols) and log it.
2. **Interactive Status LCD**: Add a 16x2 I2C LCD (Project 58) that displays the name of the pressed remote button.
3. **Buzzer Feedback**: Add a buzzer on GPIO 4 that plays a short tick sound for every command successfully decoded.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Signals do not decode or show as UNKNOWN | Fluorescent light interference or wrong library version | Keep sensor shielded from direct ambient bulb light; check if using `IRremote` v3.x or higher |
| Serial Monitor prints nothing | Pin configuration mismatch | Double-check OUT pin is wired to GPIO 15 |
| Repeating commands trigger multiple actions | Missing `IrReceiver.resume()` call | Ensure `IrReceiver.resume()` is called at the end of the decode block |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [118 - ESP32 IR Remote LED Control](118-esp32-ir-remote-led-control.md)
- [89 - ESP32 HC-05 Bluetooth Serial UART Link](89-esp32-hc05-bluetooth-serial-uart-link.md)
- [112 - ESP32 DS3231 Real Time Clock Display](112-esp32-ds3231-real-time-clock-display.md)
