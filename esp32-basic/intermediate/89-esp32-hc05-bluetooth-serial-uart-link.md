# 89 - ESP32 HC-05 Bluetooth Serial UART Link

Bridge communication between the ESP32 Serial Monitor and an HC-05 Bluetooth module, creating a wireless UART serial data pass-through.

## Goal
Learn how to use multiple hardware UART serial ports on the ESP32 (`Serial2`), configure baud rates, and pass text data bidirectionally between USB and Bluetooth.

## What You Will Build
An HC-05 Bluetooth module is connected to the ESP32's `Serial2` UART interface (RX2: GPIO 16, TX2: GPIO 17). The ESP32 acts as a bridge: characters received from the USB Serial Monitor are sent wirelessly to Bluetooth, and messages received over Bluetooth are printed to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth_hc05` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 Module | VCC | 5V (Vin) | Red | Power supply (5V recommended) |
| HC-05 Module | GND | GND | Black | Ground reference |
| HC-05 Module | TXD (Transmit) | GPIO16 (RX2) | Green | HC-05 Tx connects to ESP32 Rx |
| HC-05 Module | RXD (Receive) | GPIO17 (TX2) | Yellow | HC-05 Rx connects to ESP32 Tx (Use voltage divider) |

> **Wiring tip:** The HC-05 RXD pin expects 3.3V logic. While the ESP32 TX2 pin outputs 3.3V (which is safe), if you are using a 5V microcontroller in another design you would need a divider. For the ESP32, direct connection of TX2 (17) to RXD is safe, but using a 1 kΩ series resistor is a good safety habit.

## Code
```cpp
// HC-05 Bluetooth Serial UART Link
// Uses Hardware Serial2 on ESP32 (pins 16 and 17)

#define RX2_PIN 16
#define TX2_PIN 17

void setup() {
  // Initialize USB Serial Monitor
  Serial.begin(115200);
  Serial.println("ESP32 USB Serial Monitor initialised.");
  
  // Initialize Hardware Serial2 for the HC-05 Bluetooth module
  // Default baud rate for HC-05 in communication mode is 9600
  Serial2.begin(9600, SERIAL_8N1, RX2_PIN, TX2_PIN);
  Serial.println("Serial2 (Bluetooth) initialised at 9600 baud.");
  Serial.println("Ready to bridge data. Type a message...");
}

void loop() {
  // Read from Bluetooth (Serial2) and write to USB Serial Monitor (Serial)
  if (Serial2.available()) {
    char inChar = Serial2.read();
    Serial.write(inChar); // Pass to USB
  }

  // Read from USB Serial Monitor (Serial) and write to Bluetooth (Serial2)
  if (Serial.available()) {
    char outChar = Serial.read();
    Serial2.write(outChar); // Pass to Bluetooth
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **HC-05 Bluetooth Module** onto the canvas.
2. Wire HC-05 TXD to **GPIO16 (RX2)** and RXD to **GPIO17 (TX2)**.
3. Paste the code and click **Run**.
4. Type messages in the virtual Bluetooth console input and watch them appear in the Serial Monitor.

## Expected Output
Serial Monitor:
```
ESP32 USB Serial Monitor initialised.
Serial2 (Bluetooth) initialised at 9600 baud.
[Bluetooth Received]: Hello ESP32
[Sent to Bluetooth]: Hello Phone
```

## Expected Canvas Behavior
* Characters typed in the Serial Monitor input bar are immediately forwarded to the Bluetooth module widget tx stream.
* Characters sent by the Bluetooth widget rx stream appear in the Serial Monitor.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `Serial2.begin(9600, ...)` | Initializes UART2 with 9600 baud, 8 data bits, no parity, 1 stop bit, on pins 16 and 17. |
| `Serial2.available()` | Checks if bytes have arrived in the RX buffer of the second UART hardware port. |
| `Serial.write(inChar)` | Forwards the raw byte from the Bluetooth RX buffer directly to the USB connection. |

## Hardware & Safety Concept: Hardware UART vs Software Serial
Microcontrollers have dedicated hardware peripheral blocks called **UARTs** (Universal Asynchronous Receiver-Transmitter) to handle serial transmission. Hardware UARTs use dedicated shift registers and FIFO buffers in silicon, freeing up the CPU from manually timing individual bits. The ESP32 contains 3 hardware UART blocks (`Serial`, `Serial1`, and `Serial2`). Using Hardware UARTs is far more reliable at high baud rates and consumes fewer CPU cycles than "Software Serial" emulation libraries.

## Try This! (Challenges)
1. **Command Parser Mode**: Watch the incoming Bluetooth stream. If the string "LED_ON" is received, turn on the built-in LED (GPIO 2).
2. **Baud Rate Scanner**: Build a setup that tries connecting at 9600, 38400, and 115200 to find the correct baud rate of your physical HC-05 module automatically.
3. **RSSI Level Display**: (If supported by AT commands) retrieve and print the signal strength indicator.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| No communication in either direction | Tx and Rx lines crossed | Verify TXD of HC-05 connects to RX2 (GPIO 16) and RXD connects to TX2 (GPIO 17) |
| Characters are garbled or question marks | Baud rate mismatch | Verify the HC-05 is running at 9600 baud (or check if it was set to 38400 in AT mode) |
| Bluetooth connection drops out | VCC connected to 3.3V | Connect HC-05 VCC to the 5V (Vin) pin. The module regulator needs >4.5V to operate |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [90 - ESP32 Bluetooth LED Controller](90-esp32-hc05-bluetooth-led-controller-string-parsing.md)
- [91 - ESP32 Bluetooth Buzzer Tone Player](91-esp32-hc05-bluetooth-buzzer-tone-player.md)
- [92 - ESP32 Bluetooth Relay Switcher](92-esp32-hc05-bluetooth-relay-switcher.md)
