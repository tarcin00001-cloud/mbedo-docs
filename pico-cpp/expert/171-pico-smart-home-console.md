# 171 - Pico Smart Home Console

Build a smart home terminal console that manages HVAC and lighting, reports multi-sensor climates, and receives Bluetooth commands.

## Goal
Learn how to interface multiple high-load devices (relays), read climate sensors (DHT22, BMP180), parse Bluetooth strings, and drive indicator LEDs and warning buzzers.

## What You Will Build
An expert-level home automation terminal:
- **HC-05 Bluetooth Module (GP0, GP1)**: Toggles devices and streams sensor logs.
- **Relay Module (GP10)**: Toggles the home heating loop.
- **DHT22 Climate Sensor (GP12)**: Measures living room temperature and humidity.
- **BMP180 Barometer (GP4, GP5)**: Measures barometric air pressure.
- **Active Buzzer (GP14)**: Sounds alarms if safety thresholds are exceeded.
- **Green LED (GP15) & Red LED (GP16)**: Displays safety and lighting status.
- **16x2 I2C LCD (GP4, GP5)**: Displays room status (e.g., Temp, Humid, Pressure, and Light status).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| Relay Module | `relay` | Yes | Yes |
| DHT22 Climate Sensor | `dht22` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| LEDs (Green & Red) | `led` | Yes (two LEDs) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| Relay Module | IN | GP10 | Heating control switch |
| DHT22 | SDA | GP12 | Climate sensor (requires 10k pull-up) |
| BMP180 Sensor | SDA / SCL | GP4 / GP5 | Shared I2C sensor bus |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C display bus |
| Active Buzzer | VCC (+) | GP14 | Alarm chime pin |
| Green LED | Anode | GP15 | System OK light |
| Red LED | Anode | GP16 | Warning/Active light |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_BMP085.h>
#include <LiquidCrystal_I2C.h>

const int RELAY_PIN = 10;
const int DHT_PIN   = 12;
const int BUZZ_PIN  = 14;
const int GREEN_LED = 15;
const int RED_LED   = 16;

#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);
Adafruit_BMP085 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);

bool heatActive = false;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART
  dht.begin();
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZ_PIN, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);
  pinMode(RED_LED, OUTPUT);

  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZ_PIN, LOW);
  digitalWrite(GREEN_LED, HIGH); // System active
  digitalWrite(RED_LED, LOW);

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  lcd.init();
  lcd.backlight();
  
  if (bmp.begin()) {
    lcd.setCursor(0, 0);
    lcd.print("Smart Console");
    lcd.setCursor(0, 1);
    lcd.print("Gateway Online  ");
    Serial1.println("Home Automation Gateway Active.");
  } else {
    lcd.setCursor(0, 0);
    lcd.print("BMP180 Error!");
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
      digitalWrite(RED_LED, HIGH);
      Serial1.println("HVAC Heating: ON");
      tone(BUZZ_PIN, 1000, 100);
    } 
    else if (cmd == '0') {
      heatActive = false;
      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(RED_LED, LOW);
      Serial1.println("HVAC Heating: OFF");
      tone(BUZZ_PIN, 800, 100);
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
    digitalWrite(RED_LED, LOW);
    Serial1.println("WARNING: Overheat shutdown activated!");
    tone(BUZZ_PIN, 150, 1000);
  }

  // Update LCD Screen
  if (!isnan(temp) && !isnan(humid)) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("T:");
    lcd.print(temp, 1);
    lcd.print("C H:");
    lcd.print(humid, 0);
    lcd.print("%");

    lcd.setCursor(0, 1);
    if (heatActive) { lcd.print("HEAT: ON "); } else { lcd.print("HEAT: OFF"); }
    lcd.print(" P:");
    lcd.print(pressure / 100);
    lcd.print("hPa");
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-05**, **Relay**, **DHT22**, **BMP180**, **Active Buzzer**, **two LEDs**, and **I2C LCD** onto the canvas.
2. Connect HC-05 to **GP0/GP1**, Relay to **GP10**, DHT22 to **GP12**, Buzzer to **GP14**, LEDs to **GP15/GP16**, and LCD/BMP180 to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Send commands over Bluetooth (e.g. `'1'` or `'T'`) and watch the relay toggle and telemetry print.

## Expected Output

Terminal (Bluetooth):
```
Home Automation Gateway Active.
HVAC Heating: ON
Temp: 24.50C | Humid: 50.20% | Pres: 1013hPa
```

## Expected Canvas Behavior
* Startup: LCD reads `Smart Console` / `Gateway Online`. Green LED turns ON.
* Send `'1'` on BT: Relay closes, Red LED turns ON. LCD shows `HEAT: ON`.
* Send `'0'` on BT: Relay opens, Red LED turns OFF. LCD shows `HEAT: OFF`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `temp > 32.0 && heatActive` | Overheat safety checker, forcing heater cutoff and warning chimes if target comfort bounds are exceeded. |

## Hardware & Safety Concept: HVAC Overheat Cut-Offs
HVAC heating systems must implement independent thermal cut-offs. If a software control loop stalls or a relay contact welds shut, temperature can rise to dangerous levels. Security standards require adding a secondary mechanical thermal fuse in series with the heater load to cut power physically if temperature limits are breached.

## Try This! (Challenges)
1. **Critical Humidity Warning**: Sound a pulsing alarm if humidity levels exceed 85% to warn of mildew risks.
2. **Auto Backlight Toggle**: Turn OFF the LCD backlight automatically if no Bluetooth commands are received for 10 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Console stops updating LCD | Sensor polling rate too high | DHT22 sensors require at least 2 seconds between reads. Ensure the main loop delay matches climate update windows. |

## Mode Notes
This multi-device I2C and UART project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [126 - Pico Home Automation](../advanced/126-pico-home-automation.md)
- [151 - Pico Home Automation OLED](../advanced/151-pico-home-automation-oled.md)
- [161 - Pico Home Automation DHT Logger](../advanced/161-pico-home-automation-dht-logger.md)
