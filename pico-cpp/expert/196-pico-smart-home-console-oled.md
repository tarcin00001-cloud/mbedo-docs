# 196 - Pico Smart Home Console OLED

Build a smart home terminal console that manages HVAC heating relays, displays indoor climate datasets on an OLED, and receives Bluetooth commands.

## Goal
Learn how to control high-power relays, read climate sensors (DHT22, BMP180), design graphical OLED dashboards, and parse Bluetooth serial input data.

## What You Will Build
An expert-level home automation terminal HUD:
- **HC-05 Bluetooth Module (GP0, GP1)**: Toggles devices and streams sensor logs.
- **Relay Module (GP10)**: Toggles the heating system loop.
- **DHT22 Climate Sensor (GP12)**: Measures room temperature and humidity.
- **BMP180 Barometer (GP4, GP5)**: Measures barometric air pressure.
- **SSD1306 OLED (GP4, GP5)**: Displays room status (e.g. Temp, Humid, Pressure, and Heater status).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| Relay Module | `relay` | Yes | Yes |
| DHT22 Climate Sensor | `dht22` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| Relay Module | IN | GP10 | Heating control switch |
| DHT22 | SDA | GP12 | Climate sensor (requires 10k pull-up) |
| BMP180 Sensor | SDA / SCL | GP4 / GP5 | Shared I2C sensor bus |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C display bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_BMP085.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int RELAY_PIN = 10;
const int DHT_PIN   = 12;

#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);
Adafruit_BMP085 bmp;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

bool heatActive = false;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART
  dht.begin();
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start heater OFF

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
  
  if (bmp.begin()) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(10, 20);
    display.print("Smart Console");
    display.setCursor(10, 35);
    display.print("Online & Active");
    display.display();
    Serial1.println("Home Automation Gateway Active.");
  } else {
    display.clearDisplay();
    display.setCursor(10, 20);
    display.print("BMP180 Error!");
    display.display();
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
      heatActive = true;
      digitalWrite(RELAY_PIN, HIGH);
      Serial1.println("HVAC Heating: ON");
    } 
    else if (cmd == '0') {
      heatActive = false;
      digitalWrite(RELAY_PIN, LOW);
      Serial1.println("HVAC Heating: OFF");
    } 
    else if (cmd == 'T') {
      if (!isnan(temp)) {
        Serial1.print("Temp: "); Serial1.print(temp); Serial1.print("C | ");
        Serial1.print("Humid: "); Serial1.print(humid); Serial1.print("% | ");
        Serial1.print("Pres: "); Serial1.print(pressure / 100); Serial1.println("hPa");
      }
    }
  }

  // Safety cut-off: Turn heater OFF if temperature exceeds 32°C
  if (!isnan(temp) && temp > 32.0 && heatActive) {
    heatActive = false;
    digitalWrite(RELAY_PIN, LOW);
    Serial1.println("WARNING: Overheat shutdown activated!");
  }

  // Update OLED Screen
  if (!isnan(temp) && !isnan(humid)) {
    display.clearDisplay();

    // Draw header bar
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(15, 3);
    display.print("SMART CONSOLE HUD");

    // Display values
    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);

    // Row 1: Temp & Humidity
    display.setCursor(8, 20);
    display.print("T: ");
    display.print(temp, 1);
    display.print("C | H: ");
    display.print(humid, 0);
    display.print("%");

    // Row 2: Pressure
    display.setCursor(8, 34);
    display.print("Pres: ");
    display.print(pressure / 100);
    display.print(" hPa");

    // Row 3: Heater status
    display.setCursor(8, 48);
    display.print("Heater: ");
    display.print(heatActive ? "ACTIVE/ON" : "STANDBY/OFF");

    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-05**, **Relay**, **DHT22**, **BMP180**, and **SSD1306 OLED** onto the canvas.
2. Connect HC-05 to **GP0/GP1**, Relay to **GP10**, DHT22 to **GP12**, and BMP180/OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Send commands over Bluetooth (e.g. `'1'` or `'T'`) and watch the relay toggle and telemetry display on the OLED.

## Expected Output

Terminal (Bluetooth):
```
Home Automation Gateway Active.
HVAC Heating: ON
Temp: 24.50C | Humid: 50.20% | Pres: 1013hPa
```

## Expected Canvas Behavior
* Startup: OLED reads `SMART CONSOLE HUD`. Relay is OFF.
* Send `'1'` on BT: Relay closes. OLED heater row reads `Heater: ACTIVE/ON`.
* Send `'0'` on BT: Relay opens. OLED heater row reads `Heater: STANDBY/OFF`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `temp > 32.0 && heatActive` | Overheat safety checker, forcing heater cutoff and warning reports if target comfort bounds are exceeded. |

## Hardware & Safety Concept: HVAC Overheat Cut-Offs
HVAC heating systems must implement independent thermal cut-offs. If a software control loop stalls or a relay contact welds shut, temperature can rise to dangerous levels. Security standards require adding a secondary mechanical thermal fuse in series with the heater load to cut power physically if temperature limits are breached.

## Try This! (Challenges)
1. **Critical Humidity Warning**: Connect a buzzer on GP14 and sound a beep if humidity exceeds 85%.
2. **Display Sleep**: Turn OFF the OLED screen automatically if no Bluetooth commands are received for 15 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Console stops updating OLED | Sensor polling rate too high | DHT22 sensors need a 2-second sleep window between samples. Ensure the update speed in loop allows settling. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [126 - Pico Home Automation](../advanced/126-pico-home-automation.md)
- [151 - Pico Home Automation OLED](../advanced/151-pico-home-automation-oled.md)
- [161 - Pico Home Automation DHT Logger](../advanced/161-pico-home-automation-dht-logger.md)
- [171 - Pico Smart Home Console](171-pico-smart-home-console.md)
