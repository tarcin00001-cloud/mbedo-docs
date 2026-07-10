# 119 - BT Environment Monitor

Integrate an HC-05 Bluetooth module, a DHT22 temperature/humidity sensor, an MQ-2 gas sensor, and a character LCD to create a remote query-response system. A user can request telemetry values wirelessly from their device.

## Goal
Learn how to read incoming wireless commands via SoftwareSerial, parse query characters, retrieve sensor data from analog and single-wire devices, and write custom telemetry packets back to the Bluetooth buffer.

## What You Will Build
A smart home sensor node:
- **BT Query Parsing**: Listens for wireless command characters:
  - `t` or `T`: Queries temperature and humidity.
  - `g` or `G`: Queries gas level.
- **Wireless Response**: Responds via `BT.println()` with structured data (e.g. `Temp: 24.5C, Hum: 60%`).
- **LCD Console**: Displays current local telemetry status and shows the last processed command.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-05 Bluetooth | `hc05_bluetooth` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| MQ-2 Gas Sensor | `mq2` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-05 Bluetooth | VCC | 5V | Power supply |
| HC-05 Bluetooth | TXD | D10 | SoftwareSerial RX (Ardu RX ← HC TX) |
| HC-05 Bluetooth | RXD | D11 | SoftwareSerial TX (Ardu TX → HC RX) |
| HC-05 Bluetooth | GND | GND | Ground reference |
| DHT22 Sensor | VCC | 5V | Power supply |
| DHT22 Sensor | SDA | D2 | Digital data |
| DHT22 Sensor | GND | GND | Ground reference |
| MQ-2 Gas Sensor | VCC | 5V | Power supply |
| MQ-2 Gas Sensor | AO | A1 | Analog gas level |
| MQ-2 Gas Sensor | GND | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <SoftwareSerial.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

const int BT_RX_PIN  = 10;
const int BT_TX_PIN  = 11;
const int DHT_PIN    = 2;
const int DHT_TYPE   = DHT22;
const int GAS_PIN    = A1;

SoftwareSerial BT(BT_RX_PIN, BT_TX_PIN);
DHT dht(DHT_PIN, DHT_TYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  BT.begin(9600);
  dht.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.print("BT Sensor Node");
  delay(1500);
  lcd.clear();

  Serial.println("BT Environment Monitor Active");
}

void loop() {
  float humidity = dht.readHumidity();
  float tempC = dht.readTemperature();
  int gasLevel = analogRead(GAS_PIN);

  // Local update on LCD
  lcd.setCursor(0, 0);
  lcd.print("T:");
  lcd.print(tempC, 0);
  lcd.print("C H:");
  lcd.print(humidity, 0);
  lcd.print("% G:");
  lcd.print(gasLevel);
  lcd.print("   ");

  if (BT.available()) {
    char cmd = BT.read();
    Serial.print("Received BT Command: ");
    Serial.println(cmd);

    lcd.setCursor(0, 1);
    lcd.print("Cmd: ");
    lcd.print(cmd);
    lcd.print(" Processing ");

    if (cmd == 't' || cmd == 'T') {
      BT.print("TEMP:");
      BT.print(tempC, 1);
      BT.print("C,HUM:");
      BT.print(humidity, 1);
      BT.println("%");
      Serial.println("Sent Temp/Hum data via BT");
    } else if (cmd == 'g' || cmd == 'G') {
      BT.print("GAS:");
      BT.println(gasLevel);
      Serial.println("Sent Gas data via BT");
    } else {
      BT.println("ERR: Unknown Cmd");
      Serial.println("Sent Error response via BT");
    }
    
    delay(500);
    lcd.setCursor(0, 1);
    lcd.print("System Idle     ");
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-05 Bluetooth**, **DHT22 Sensor**, **MQ-2 Gas Sensor**, and **16x2 I2C LCD** onto the canvas.
2. Connect HC-05: **VCC** to **5V**, **TXD** to **D10**, **RXD** to **D11**, **GND** to **GND**.
3. Connect DHT22: **VCC** to **5V**, **SDA** to **D2**, **GND** to **GND**.
4. Connect MQ-2 Gas: **VCC** to **5V**, **AO** to **A1**, **GND** to **GND**.
5. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
6. Paste the code into the editor, select interpreted mode, and click **Run**.
7. Open the **Bluetooth Terminal** tab in the dock. Type `T` or `G` and press send to see telemetry responses.

## Expected Output

Bluetooth Terminal:
```
Waiting for BT.print() / BT.println() output...
TEMP:24.0C,HUM:60.0%
GAS:180
```

LCD Display:
```
T:24C H:60% G:180
System Idle     
```

## Expected Canvas Behavior
| Bluetooth Input | DHT22 Temp | MQ-2 Gas Level | BT Output Packet | LCD Line 2 (Temporary) |
| --- | --- | --- | --- | --- |
| `'t'` | 30.5 C | - | `TEMP:30.5C,HUM:55.0%` | `Cmd: t Processing` |
| `'g'` | - | 420 | `GAS:420` | `Cmd: g Processing` |
| `'x'` | - | - | `ERR: Unknown Cmd` | `Cmd: x Processing` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `BT.read()` | Retrieves the oldest raw character byte sitting in the SoftwareSerial circular RX buffer. |
| `BT.print()` | Writes structured ASCII response data over the virtual TX pin back to the connected client. |
| `lcd.print("   ")` | Trailing space padding prevents residue when the length of values decreases dynamically. |

## Hardware & Safety Concept: Telemetry Query Protocols
Using query-response mechanisms rather than continuous broadcasting helps reduce power usage and mitigates radio collision issues in environments where multiple sensors share communication bands. SoftwareSerial allows the Arduino Uno to communicate with peripheral transceivers without disabling the hardware serial connection used for programming and console logging.

## Try This! (Challenges)
1. **Low Power Sleep Indicator**: Pulse an LED on pin D5 to indicate that the sensor node is in a low-power listening state when no commands are received for 10 seconds.
2. **Threshold Broadcast**: Modify the code to immediately broadcast an alarm alert (`ALARM: GAS DETECTED!`) via Bluetooth if the gas level climbs above 500, regardless of commands.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Bluetooth RX fails | RX/TX lines are swapped | Confirm Arduino pin 10 goes to HC-05 TXD, and pin 11 goes to RXD. |
| Garbage data in terminal | Speed mismatch | Verify that the SoftwareSerial initialization matches the hardware module speed (`BT.begin(9600)`). |

## Mode Notes
This Bluetooth serial query structure runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [72 - BT Serial Bridge](../intermediate/72-bt-serial-bridge.md)
- [76 - BT Sensor Report](../intermediate/76-bt-sensor-report.md)
- [87 - Air Quality Monitor](../advanced/87-air-quality-monitor.md)
