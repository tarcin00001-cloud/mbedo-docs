# 76 - BT Sensor Report

Read an analog light sensor (LDR) and transmit the light level values wirelessly over Bluetooth.

## Goal
Learn how to read an analog sensor and transmit dynamic integer values as formatted text streams over a SoftwareSerial Bluetooth link.

## What You Will Build
The Arduino reads the ambient light level from the LDR sensor on pin A0. Every 1 second, it transmits the formatted light level reading (e.g., "LDR Value: 512") over the Bluetooth serial channel.

**Why A0, D2, and D3?** Pin A0 measures the analog light voltage. Pins D2/D3 manage the Bluetooth serial transmission.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05_bluetooth` | Yes | Yes |
| LDR Light Sensor | `ldr` | Yes | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-05 Module | TXD | D2 | Software RX |
| HC-05 Module | RXD | D3 | Software TX (Use divider on hardware) |
| HC-05 Module | VCC | 5V | Power supply |
| HC-05 Module | GND | GND | Ground reference |
| LDR Sensor | VCC | 5V | Power supply |
| LDR Sensor | AO | A0 | Analog signal connection |
| LDR Sensor | GND | GND | Ground reference |

## Code
```cpp
#include <SoftwareSerial.h>

SoftwareSerial BT(2, 3); // RX = D2, TX = D3

const int LDR_PIN = A0;

void setup() {
  // Start Bluetooth serial communication
  BT.begin(9600);
  
  // Start USB serial monitor for debugging
  Serial.begin(9600);
  Serial.println("Bluetooth Light Station Active");
}

void loop() {
  int ldrVal = analogRead(LDR_PIN);
  
  // Print to local USB Serial Monitor for debugging
  Serial.print("Sensor Readout: ");
  Serial.println(ldrVal);
  
  // Transmit wirelessly over Bluetooth
  BT.print("LDR Value: ");
  BT.println(ldrVal);
  
  delay(1000); // Send data every 1 second
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-05 Bluetooth Module**, and **LDR Light Sensor** onto the canvas.
2. Connect HC-05: **TXD** to **D2**, **RXD** to **D3**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect LDR: **VCC** to **5V**, **AO** to **A0**, and **GND** to **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the LDR sensor, adjust its light slider, and watch the values print in the Terminal.

## Expected Output

Terminal:
```
Bluetooth Light Station Active
Sensor Readout: 450
Sensor Readout: 780
...
```
The virtual Bluetooth device receives:
```
LDR Value: 450
LDR Value: 780
...
```

### Expected Canvas Behavior

| LDR Light Input Slider | Pin A0 Reading | Transmitted BT Output |
| --- | --- | --- |
| Dark (20%) | ~200 | "LDR Value: 200" |
| Bright (80%) | ~800 | "LDR Value: 800" |

The Bluetooth transmission updates dynamically to match the LDR slider.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `BT.print("LDR Value: ")` | Sends the label prefix over the Bluetooth serial channel. |
| `BT.println(ldrVal)` | Converts the integer sensor reading to characters and transmits it, followed by a newline byte. |

## Hardware & Safety Concept: Wireless Data Logging
Wireless data loggers are crucial in agricultural telemetry, meteorological stations, and industrial monitoring. Transmitting raw sensor values at timed intervals allows central receivers or smartphone apps to log parameters over time, plot real-time charts, and store records without physical wired connections.

## Try This! (Challenges)
1. **Dynamic Scaling**: Use the `map()` function to convert the 0-1023 LDR reading to a 0-100% scale before transmitting over Bluetooth.
2. **Threshold Alert**: Only transmit data over Bluetooth if the light level changes by more than `50` units from the last reading, conserving power.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Terminal prints garbage characters | Baud rate mismatch | Verify that both `Serial.begin(9600)` and `BT.begin(9600)` match the settings in your Terminal window. |
| LDR readings stay static | Divider wiring error | Verify the LDR AO pin connects to Arduino pin A0. |

## Mode Notes
These patterns (analog reads combined with SoftwareSerial print transmissions) are supported by MbedO interpreted mode.

## Related Projects
- [18 - Light Meter](../beginner/18-light-meter.md)
- [72 - BT Serial Bridge](72-bt-serial-bridge.md)
- [77 - BT Distance Report](77-bt-distance-report.md)
