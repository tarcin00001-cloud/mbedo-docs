# 161 - Pico Home Automation DHT Logger

Build a wireless smart home climate gateway hub that controls lighting relays, monitors weather conditions, and streams CSV data logs over Bluetooth.

## Goal
Learn how to parse Bluetooth commands (UART), read multi-sensor datasets (DHT22 climate, BMP180 barometric pressure), and output structured CSV logs wirelessly.

## What You Will Build
A smart home gateway hub:
- **HC-05 Bluetooth Module (GP0, GP1)**: Receives commands and streams logs.
- **Relay Module (GP10)**: Toggles living room lighting.
- **DHT22 Climate Sensor (GP12)**: Reads room temperature and humidity.
- **BMP180 Barometer (GP4, GP5)**: Reads room air pressure.
- **16x2 I2C LCD (GP4, GP5)**: Displays light status and climate indices.
- **Wireless Telemetry**: Streams CSV logs (e.g. `Light,Temp,Hum,Pres`) over Bluetooth every 4 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| Relay Module | `relay` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| Relay Module | IN | GP10 | Lighting control |
| DHT22 | SDA | GP12 | Climate sensor |
| BMP180 Sensor | SDA / SCL | GP4 / GP5 | Shared I2C sensor bus |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C display bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_BMP085.h>
#include <LiquidCrystal_I2C.h>

const int RELAY_PIN = 10;
const int DHT_PIN   = 12;
#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);
Adafruit_BMP085 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);

bool lightState = false;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART
  dht.begin();
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start light OFF

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  lcd.init();
  lcd.backlight();
  
  if (bmp.begin()) {
    lcd.setCursor(0, 0);
    lcd.print("Smart Home Gateway");
    lcd.setCursor(0, 1);
    lcd.print("Logger Online   ");
    
    // Print CSV Header over Bluetooth
    Serial1.println("Light_status,Temp_C,Hum_pct,Pres_hPa");
  } else {
    lcd.setCursor(0, 0);
    lcd.print("BMP180 Error!   ");
    while (1);
  }
  delay(1500);
}

void loop() {
  float temp = dht.readTemperature();
  float humid = dht.readHumidity();
  long pressure = bmp.readPressure();

  // Process Bluetooth commands
  if (Serial1.available()) {
    char cmd = Serial1.read();

    if (cmd == '1') {
      lightState = true;
      digitalWrite(RELAY_PIN, HIGH);
      Serial1.println("#LIGHT:ON");
    } 
    else if (cmd == '0') {
      lightState = false;
      digitalWrite(RELAY_PIN, LOW);
      Serial1.println("#LIGHT:OFF");
    }
  }

  if (!isnan(temp) && !isnan(humid)) {
    // 1. Update LCD
    lcd.clear();
    lcd.setCursor(0, 0);
    if (lightState) { lcd.print("L: ON "); } else { lcd.print("L: OFF"); }
    lcd.print(" | T: ");
    lcd.print(temp, 1);
    lcd.print("C");

    lcd.setCursor(0, 1);
    lcd.print("H: ");
    lcd.print(humid, 0);
    lcd.print("% | P: ");
    lcd.print(pressure / 100);
    lcd.print("hPa");

    // 2. Stream CSV Logs over Bluetooth
    Serial1.print(lightState ? 1 : 0);
    Serial1.print(",");
    Serial1.print(temp, 2);
    Serial1.print(",");
    Serial1.print(humid, 2);
    Serial1.print(",");
    Serial1.println(pressure / 100);
  }

  delay(4000); // 4-second update rate
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-05**, **Relay**, **DHT22**, **BMP180**, and **I2C LCD** onto the canvas.
2. Connect HC-05 to **GP0/GP1**, Relay to **GP10**, DHT22 to **GP12**, and BMP180/LCD to **GP4/GP5** in parallel.
3. Paste code, select the interpreted mode, and click **Run**.
4. Type `'1'` in the Bluetooth terminal to switch the light relay, and watch the CSV logs stream.

## Expected Output

Terminal (Bluetooth):
```
Light_status,Temp_C,Hum_pct,Pres_hPa
0,24.50,50.20,1013
#LIGHT:ON
1,24.60,50.30,1013
```

## Expected Canvas Behavior
* Press `'1'` on BT: Relay closes. LCD reads `L: ON  | T: 24.5C` / `H: 50% | P: 1013hPa`.
* CSV Output: Transmits telemetry coordinates every 4 seconds.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.print(lightState ? 1 : 0)` | Writes the lighting relay state as a 1 or 0 in the CSV data stream. |

## Hardware & Safety Concept: Bidirectional Telemetry
Smart home gateways are bidirectional: they receive control commands (such as lighting toggles) and transmit sensor telemetry (such as room temperature) back to the user. To prevent transmission errors, real smart home platforms wrap serial streams in structured packages (like JSON or MQTT topics) containing validation checksums.

## Try This! (Challenges)
1. **Critical Overheat Alarm**: Sound a buzzer on GP14 if the temperature exceeds 35°C, and send a warning message over Bluetooth.
2. **Auto Light Timeout**: Automatically turn the light relay OFF after 10 seconds of inactivity to save energy.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Commands do not register | Baud rate mismatch | Ensure the smartphone Bluetooth terminal app is set to 9600 baud rate to match `Serial1.begin(9600)`. |

## Mode Notes
This multi-device I2C and UART project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [126 - Pico Home Automation](126-pico-home-automation.md)
- [129 - Pico Weather Logger](129-pico-weather-logger.md)
- [151 - Pico Home Automation OLED](151-pico-home-automation-oled.md)
