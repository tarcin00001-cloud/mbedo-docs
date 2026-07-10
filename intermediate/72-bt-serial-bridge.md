# 72 - BT Serial Bridge

Create a communication bridge to send and receive text between the Arduino USB Serial Monitor and an HC-05 Bluetooth module.

## Goal
Learn how to use the `SoftwareSerial.h` library to establish a secondary serial port on digital pins, creating a transparent bridge with the hardware USB port.

## What You Will Build
The Arduino polls both the hardware Serial port (USB) and the software-simulated Bluetooth serial port (pins D2/D3). Any character typed in the Terminal is forwarded to the Bluetooth module, and any character received over Bluetooth is printed to the Terminal.

**Why D2 and D3?** Pins D2 (RX) and D3 (TX) are configured as software-driven serial lines to communicate with the Bluetooth module's UART pins, leaving the hardware Serial (D0/D1) free for USB debug printing.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05_bluetooth` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-05 Module | TXD | D2 | Transmit connects to Software RX |
| HC-05 Module | RXD | D3 | Receive connects to Software TX (See note) |
| HC-05 Module | VCC | 5V | Power supply (5V) |
| HC-05 Module | GND | GND | Ground reference |

> **Hardware Voltage Warning:** The HC-05 RXD pin operates on **3.3V logic levels**. When wiring to a physical 5V Arduino Uno, you must use a **voltage divider** (e.g. a 1k ohm and 2k ohm resistor) on the Arduino TX (pin D3) line to step down the 5V signal to 3.3V, preventing damage to the Bluetooth module receiver.

## Code
```cpp
#include <SoftwareSerial.h>

// Define SoftwareSerial pins: RX = pin 2, TX = pin 3
SoftwareSerial BT(2, 3);

void setup() {
  // Start hardware serial for USB terminal
  Serial.begin(9600);
  Serial.println("USB Serial Port Active");
  
  // Start software serial for Bluetooth
  BT.begin(9600);
  Serial.println("HC-05 Bluetooth Bridge Active. Start typing...");
}

void loop() {
  // 1. Read from Bluetooth, print to Serial
  if (BT.available()) {
    char c = BT.read();
    Serial.print(c);
  }
  
  // 2. Read from Serial, print to Bluetooth
  if (Serial.available()) {
    char c = Serial.read();
    BT.print(c);
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **HC-05 Bluetooth Module** onto the canvas.
2. Connect HC-05 **TXD** to Arduino **D2**, **RXD** to Arduino **D3**, **VCC** to Arduino **5V**, and **GND** to Arduino **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Type characters into the Terminal text prompt, hit Enter, and watch them echo through the virtual Bluetooth transceiver.

## Expected Output
Terminal:
```
USB Serial Port Active
HC-05 Bluetooth Bridge Active. Start typing...
Hello from PC! (typed in serial monitor)
Hello from Phone! (received from virtual Bluetooth device)
```

### Expected Canvas Behavior

| Action | Signal Flow | Expected Behavior |
| --- | --- | --- |
| Type in Terminal | Serial -> BT | Transmits data to virtual HC-05 RXD pin |
| Send from Phone | BT -> Serial | Transmits data from virtual HC-05 TXD pin to D2 |

All sent and received text streams are forwarded transparently.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `SoftwareSerial BT(2, 3)` | Creates a virtual UART serial port named `BT` with pin D2 as Receiver (RX) and pin D3 as Transmitter (TX). |
| `BT.begin(9600)` | Starts serial communications at 9600 baud rate on the virtual port. |
| `BT.available()` | Checks if any bytes have been received over the air and are waiting in the memory buffer. |

## Hardware & Safety Concept: Logic Level Translation
Many modern wireless sensors (Bluetooth, WiFi, GPS) operate on **3.3V logic**, meaning a HIGH signal is 3.3V. The Arduino Uno operates on **5V logic**, outputting 5V on its digital pins. 
- While a 3.3V TX signal from the Bluetooth module is high enough to be detected by the Arduino Uno RX pin, a 5V TX signal from the Arduino Uno can overload the internal protection diodes of the Bluetooth module's RXD pin.
- To prevent wear and overheating, a simple resistor voltage divider is placed on the line to drop the voltage safely.

## Try This! (Challenges)
1. **Startup Check**: Add a verification command: print "Ready to pair..." over Bluetooth `BT.println("Ready to pair...")` in `setup()`.
2. **Speed Switch**: Change the baud rates to 38400 (which is the default baud rate for configuring the HC-05 using AT commands).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Warning: "no SoftwareSerial BT declaration detected" | Incorrect object variable name | Ensure your SoftwareSerial variable is declared exactly as `BT`: `SoftwareSerial BT(2, 3);`. |
| Garbage characters in Terminal | Baud rate mismatch | Ensure both the hardware `Serial.begin(9600)` and Software `BT.begin(9600)` rates match the baud rate settings in the Terminal window dropdown. |

## Mode Notes
These transparent echo patterns are parsed and supported by MbedO interpreted mode.

## Related Projects
- [04 - Serial Print](../beginner/04-serial-print.md)
- [73 - BT LED Control](73-bt-led-control.md)
