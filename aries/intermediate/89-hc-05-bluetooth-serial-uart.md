# 89 - HC-05 Bluetooth Serial UART Link

Bridge communication between the ARIES USB Serial Monitor and an HC-05 Bluetooth module, creating a wireless UART serial data pass-through.

## Goal
Learn how to use multiple hardware UART serial ports on the ARIES v3 board (`Serial2`), configure baud rates, and pass text data bidirectionally between USB and Bluetooth without using loops inside C++ code.

## What You Will Build
An HC-05 Bluetooth module is connected to the ARIES v3 board's `Serial2` UART interface (GPIO 16 and 17). The board acts as a bridge: characters received from the USB Serial Monitor are sent wirelessly to Bluetooth, and messages received over Bluetooth are printed to the USB Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth_hc05` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 Module | VCC | 5V | Red | Power supply (5V recommended) |
| HC-05 Module | GND | GND | Black | Ground reference |
| HC-05 Module | TXD (Transmit) | GPIO 16 (RX2) | Green | HC-05 Tx connects to ARIES Rx2 |
| HC-05 Module | RXD (Receive) | GPIO 17 (TX2) | Yellow | HC-05 Rx connects to ARIES Tx2 |

> **Wiring tip:** The HC-05 RXD pin expects 3.3V logic. While the ARIES TX2 pin outputs 3.3V (which is safe), if you are using a 5V microcontroller in another design you would need a divider. For the ARIES board, direct connection of TX2 (17) to RXD is safe, but using a 1 kΩ series resistor is a good safety habit.

## Code
```cpp
// HC-05 Bluetooth Serial UART Link
// Uses Hardware Serial2 on ARIES v3 (pins 16 and 17)

void setup() {
  // Initialize USB Serial Monitor
  Serial.begin(115200);
  Serial.println("ARIES USB Serial Monitor initialized.");
  
  // Initialize Hardware Serial2 for the HC-05 Bluetooth module
  // Default baud rate for HC-05 in communication mode is 9600
  Serial2.begin(9600);
  Serial.println("Serial2 (Bluetooth) initialized at 9600 baud.");
  Serial.println("Ready to bridge data. Type a message...");
}

void loop() {
  // Read from Bluetooth (Serial2) and write to USB Serial Monitor (Serial)
  if (Serial2.available() > 0) {
    char inChar = Serial2.read();
    Serial.write(inChar); // Pass to USB
  }

  // Read from USB Serial Monitor (Serial) and write to Bluetooth (Serial2)
  if (Serial.available() > 0) {
    char outChar = Serial.read();
    Serial2.write(outChar); // Pass to Bluetooth
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** and **HC-05 Bluetooth Module** components onto the canvas.
2. Wire HC-05 TXD to **GPIO 16 (RX2)** and RXD to **GPIO 17 (TX2)**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. Type messages in the virtual Bluetooth console input and watch them appear in the Serial Monitor.

## Expected Output
Serial Monitor:
```
ARIES USB Serial Monitor initialized.
Serial2 (Bluetooth) initialized at 9600 baud.
Hello ARIES
Hello Phone
```

## Expected Canvas Behavior
* Characters typed in the Serial Monitor input bar are immediately forwarded to the Bluetooth module widget tx stream.
* Characters sent by the Bluetooth widget rx stream appear in the Serial Monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Serial2.begin(9600)` | Initializes UART2 with 9600 baud. |
| `Serial2.available() > 0` | Checks if bytes have arrived in the RX buffer of the second UART hardware port. |
| `Serial.write(inChar)` | Forwards the raw byte from the Bluetooth RX buffer directly to the USB connection. |
| `Serial2.write(outChar)` | Forwards the raw byte from the USB Serial Monitor to the Bluetooth TX queue. |

## Hardware & Safety Concept
* **Hardware UART vs. Software Serial**: Microcontrollers have dedicated hardware peripheral blocks called **UARTs** (Universal Asynchronous Receiver-Transmitter) to handle serial transmission. Hardware UARTs use dedicated shift registers and FIFO buffers in silicon, freeing up the CPU from manually timing individual bits. The ARIES v3 board contains multiple hardware UART blocks. Using Hardware UARTs is far more reliable at high baud rates and consumes fewer CPU cycles than "Software Serial" emulation libraries.

## Try This! (Challenges)
1. **Command Mode**: Parse the incoming Bluetooth stream. If the character '1' is received, turn on the built-in LED (GPIO 23). If '0' is received, turn it off.
2. **Echo Confirmation**: Modify the code so that characters received from Bluetooth are immediately echoed back to the Bluetooth transmitter in addition to the USB serial line.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| No communication in either direction | Tx and Rx lines crossed | Verify TXD of HC-05 connects to RX2 (GPIO 16) and RXD connects to TX2 (GPIO 17). |
| Characters are garbled or question marks | Baud rate mismatch | Verify the HC-05 is running at 9600 baud (or check if it was set to 38400 in AT mode). |
| Bluetooth module does not pair | Connection status pin issue | Ensure the module has sufficient power and status LEDs are flashing. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [90 - HC-05 Bluetooth LED Controller](90-hc-05-bluetooth-led-controller.md)
- [91 - HC-05 Bluetooth Buzzer Tone](91-hc-05-bluetooth-buzzer-tone.md)
- [92 - HC-05 Bluetooth Relay Switcher](92-hc-05-bluetooth-relay-switcher.md)
