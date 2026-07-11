# 120 - Bluetooth DHT22 Telemetry Broadcast

Build a wireless climate telemetry node that reads temperature and humidity from a DHT22 sensor and broadcasts the readings via an HC-05 Bluetooth module on the VEGA ARIES v3 board.

## Goal
Learn how to interface single-wire climate sensors, configure secondary hardware UART serial communication ports (`Serial2`), and transmit telemetry data packets wirelessly.

## What You Will Build
A remote environmental climate sensor beacon. The ARIES board measures temperature and humidity from a DHT22 sensor. It formats this data into a text string and transmits it wirelessly over Bluetooth using an HC-05 module connected to the hardware `Serial2` port. The data can be viewed on a paired mobile app or laptop terminal.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Climate Sensor | `dht22` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (pull-up) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | VCC | 3V3 | Red | Sensor operating voltage (3.3V) |
| DHT22 Sensor | GND | GND | Black | Ground reference |
| DHT22 Sensor | SDA (Data) | GPIO 12 | Green | Single-wire data line |
| HC-05 Module | VCC | 5V | Red | Bluetooth module operating power |
| HC-05 Module | GND | GND | Black | Ground reference |
| HC-05 Module | TXD | RX2 (GP21) | Blue | HC-05 TX to ARIES RX2 |
| HC-05 Module | RXD | TX2 (GP20) | Yellow | HC-05 RX to ARIES TX2 |

> **Wiring tip:** The HC-05 TXD connects to ARIES RX2 (GP21), and the HC-05 RXD connects to ARIES TX2 (GP20). Cross the TX/RX pins so that the transmitter on one device connects to the receiver on the other.

## Code
```cpp
#include <DHT.h>

const int DHT_PIN = 12; // DHT22 on GPIO 12
#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);

void setup() {
  // Initialize primary USB Serial for local monitoring
  Serial.begin(9600);
  
  // Initialize secondary Hardware Serial2 for Bluetooth communications
  Serial2.begin(9600); 

  dht.begin();
  
  Serial.println("Bluetooth climate broadcaster online.");
}

void loop() {
  float temp = dht.readTemperature();
  float hum = dht.readHumidity();

  // Verify that the DHT22 returned valid readings
  if (!isnan(temp) && !isnan(hum)) {
    // Broadcast telemetry wirelessly over Bluetooth Serial2
    Serial2.print("Temp:");
    Serial2.print(temp, 1);
    Serial2.print("C,Hum:");
    Serial2.print(hum, 1);
    Serial2.println("%");

    // Mirror to USB Serial for debugging
    Serial.print("Temp:");
    Serial.print(temp, 1);
    Serial.print("C | Hum:");
    Serial.print(hum, 1);
    Serial.println("%");
  } else {
    Serial.println("Sensor Read Error!");
  }

  delay(2000); // 2-second sampling interval required by DHT22
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DHT22 Climate Sensor**, and **HC-05 Bluetooth Module** onto the canvas.
2. Wire the DHT22: **VCC** to **3V3**, **GND** to **GND**, and **SDA** to **GPIO 12**.
3. Wire the HC-05: **VCC** to **5V**, **GND** to **GND**, **TXD** to **RX2 (GP21)**, and **RXD** to **TX2 (GP20)**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.
7. Observe the Serial Monitor and virtual Bluetooth terminal outputs.

## Expected Output
Serial Monitor (Local Debug):
```
Bluetooth climate broadcaster online.
Temp:24.8C | Hum:58.2%
Temp:24.9C | Hum:58.1%
```
Bluetooth Terminal (Remote App):
```
Temp:24.8C,Hum:58.2%
Temp:24.9C,Hum:58.1%
```

## Expected Canvas Behavior
* The DHT22 reads temperature and humidity from the simulated sliders.
* Changing the temperature and humidity sliders changes the output printed on the Serial Monitor and the data packages streamed out of the HC-05 RXD/TXD lines.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Serial2.begin(9600)` | Activates the second hardware UART controller on ARIES at a 9600 baud rate. |
| `dht.readTemperature()` | Samples temperature from the DHT22 sensor. |
| `!isnan(...)` | Ensures the data is valid before printing. |
| `Serial2.print(...)` | Transmits the data packet wirelessly through the Bluetooth module. |

## Hardware & Safety Concept
* **UART Cross-Wiring**: Serial UART communication uses separate transmit (TX) and receive (RX) lines. The transmitter pin of the ARIES board (TX2) must connect to the receiver pin of the Bluetooth module (RXD), and vice-versa, so that data flows in the correct direction.
* **Level Shifting on RXD**: The HC-05 module's logic level is typically 3.3V, but it can be powered by 5V. When connecting to 5V microcontrollers, you must use a voltage divider (e.g., a 1k-ohm and 2k-ohm resistor) on the HC-05 RXD line to prevent damage from 5V signals. The ARIES board operates on 3.3V logic, so it is safe to connect the pins directly.

## Try This! (Challenges)
1. **Bluetooth Command Mode**: Monitor incoming Bluetooth data using `Serial2.available()`. If the user sends the character 'R' over Bluetooth, trigger a sensor read immediately and return the values.
2. **LED Status Broadcast**: Connect an LED to GPIO 15. Allow the user to turn this LED ON or OFF by sending commands over Bluetooth (e.g. '1' to turn ON, '0' to turn OFF).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Bluetooth terminal receives garbage characters | Baud rate mismatch | Ensure both the `Serial2.begin(9600)` and your Bluetooth terminal application are set to the same speed (9600 baud). |
| Telemetry does not send to remote device | HC-05 TXD/RXD pins crossed | Check the wiring. Ensure ARIES TX2 (GP20) connects to HC-05 RXD, and ARIES RX2 (GP21) connects to HC-05 TXD. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [71 - Pico DHT Serial](../intermediate/71-pico-dht-serial.md)
- [100 - Automatic Smart Fan](100-automatic-smart-fan.md)
