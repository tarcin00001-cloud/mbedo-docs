# 89 - Pico Bluetooth Serial

Establish a wireless Bluetooth Serial communication link using an HC-05 module.

## Goal
Learn how to configure hardware UART serial ports on the Pico to transmit and receive data wirelessly.

## What You Will Build
A wireless terminal bridge:
- **HC-05 Bluetooth Module (GP0 TX, GP1 RX)**: Connected to UART0. Characters typed on a remote smartphone Bluetooth terminal are echoed back to the terminal.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-05 | VCC | 5V (or 3.3V depending on module) | Power supply |
| HC-05 | TXD | GP1 (UART0 RX) | Data from module to Pico |
| HC-05 | RXD | GP0 (UART0 TX) | Data from Pico to module (requires logic divider)|
| HC-05 | GND | GND | Ground return |

## Code
```cpp
void setup() {
  // Initialize standard hardware UART0 (default TX=GP0, RX=GP1)
  Serial1.begin(9600);
  Serial1.println("Bluetooth Bridge Online - Send Characters");
}

void loop() {
  // Check if characters are received wirelessly from Bluetooth
  if (Serial1.available()) {
    char data = Serial1.read();
    
    // Echo the character back to the Bluetooth terminal
    Serial1.print("Echo: ");
    Serial1.println(data);
  }
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **HC-05 Bluetooth Module** onto the canvas.
2. Connect HC-05: **VCC** to **5V**, **TXD** to **GP1**, **RXD** to **GP0**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Type characters in the simulated Bluetooth terminal interface and observe the echoed results.

## Expected Output

Terminal:
```
Bluetooth Bridge Online - Send Characters
Echo: A
Echo: B
```

## Expected Canvas Behavior
* Sending `A` via Bluetooth prints: `Echo: A`
* Sending `B` via Bluetooth prints: `Echo: B`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.begin(9600)` | Opens the secondary hardware UART port (`Serial1`) mapped to pins GP0 and GP1 for communication with the HC-05. |
| `Serial1.available()` | Checks if any serial bytes have arrived from the HC-05 buffer. |
| `Serial1.read()` | Retrieves the oldest byte from the internal serial receive ring buffer. |

## Hardware & Safety Concept: UART Logic Level Shifting
The HC-05 RXD pin expects 3.3V logic signals, but many modules are powered by 5V. To prevent damage to the module's RX input when using a 5V controller, or to protect the Pico's 3.3V input pins on other modules, use a **10k/20k voltage divider** on the Pico TX (GP0) to HC-05 RXD line to step down 5V signals to safe 3.3V limits.

## Try This! (Challenges)
1. **Case Changer**: Modify the code to echo characters in uppercase (e.g. sending `a` echoes back `A`).
2. **Alert Tone**: Sound a buzzer on GP14 briefly for 100 ms every time a new character is received.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| No communication in either direction | Swapped TX/RX wires | Connect Pico TX (GP0) to HC-05 RXD, and Pico RX (GP1) to HC-05 TXD. Connecting TX-to-TX and RX-to-RX is a common mistake. |

## Mode Notes
This basic serial communication project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [90 - Pico Bluetooth LED](90-pico-bt-led.md)
- [92 - Pico Bluetooth Relay](92-pico-bt-relay.md)
