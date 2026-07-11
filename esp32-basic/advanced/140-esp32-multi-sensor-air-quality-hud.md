# 140 - ESP32 Multi-Sensor Air Quality HUD

Build an air quality monitoring station that reads temperature and humidity from a DHT22, combined with gas density levels from an MQ-2 sensor, displaying status reports and safety ratings on an SSD1306 OLED screen.

## Goal
Learn how to build multi-variable safety monitors, display custom graphics warning states on OLED screens, and integrate buzzer alarms for safety.

## What You Will Build
An MQ-2 gas sensor is connected to analog GPIO 34. A DHT22 sensor is connected to GPIO 4. A buzzer is on GPIO 15, and an SSD1306 OLED screen is connected via I2C (GPIO 21/22). The screen displays climate values, raw gas density, and a safety rating (SAFE / WARNING / DANGER). If gas exceeds 1800, the buzzer sounds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MQ-2 Gas Sensor Module | `mq2` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MQ-2 Sensor | AO (Analog) | GPIO34 | Yellow | Gas density input |
| MQ-2 Sensor | VCC / GND | 5V / GND | Red / Black | Heater power rails |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Climate digital input |
| DHT22 Sensor | VCC / GND | 3V3 / GND | Red / Black | Sensor power |
| SSD1306 OLED | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| SSD1306 OLED | VCC / GND | 3V3 / GND | Red / Black | OLED power |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm horn output |
| Active Buzzer | GND (−) | GND | Black | Ground reference |

> **Wiring tip:** The MQ-2 internal heater consumes significant current (up to 150 mA). Power it from the 5V Vin pin, not from the 3.3V rail.

## Code
```cpp
// Multi-Sensor Air Quality HUD (MQ-2 + DHT22 + OLED)
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <DHT.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

const int MQ2_PIN = 34;
const int DHT_PIN = 4;
const int BUZZER_PIN = 15;

#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

// Gas safety thresholds (raw ADC 0-4095)
const int GAS_WARN_LIMIT = 1000;
const int GAS_DANGER_LIMIT = 1800;

void setup() {
  Serial.begin(115200);
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  dht.begin();
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED allocation failed!");
    while(1) {}
  }
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("Air Monitor Init...");
  display.display();
  
  delay(1500);
}

void loop() {
  // Read sensors
  float temp = dht.readTemperature();
  float hum = dht.readHumidity();
  int rawGas = analogRead(MQ2_PIN);
  
  // Determine Safety Rating
  String rating = "SAFE";
  bool alarmActive = false;
  
  if (rawGas >= GAS_DANGER_LIMIT) {
    rating = "DANGER!";
    alarmActive = true;
  } 
  else if (rawGas >= GAS_WARN_LIMIT) {
    rating = "WARNING";
  }
  
  // Actuate Buzzer
  if (alarmActive) {
    digitalWrite(BUZZER_PIN, HIGH);
  } else {
    digitalWrite(BUZZER_PIN, LOW);
  }
  
  // Update OLED Display
  display.clearDisplay();
  
  // Header
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.print("AIR MONITOR STATION");
  display.drawFastHLine(0, 10, 128, SSD1306_WHITE);
  
  // Climate variables
  display.setCursor(0, 15);
  display.print("Temp: "); 
  if (!isnan(temp)) {
    display.print(temp, 1); display.print("C");
  } else {
    display.print("Error");
  }
  
  display.setCursor(70, 15);
  display.print("Hum: ");
  if (!isnan(hum)) {
    display.print(hum, 0); display.print("%");
  } else {
    display.print("Error");
  }
  
  // Gas data
  display.setCursor(0, 30);
  display.print("Gas density: "); display.print(rawGas);
  
  // Rating banner
  display.setTextSize(2);
  display.setCursor(0, 45);
  display.print("AQI: ");
  display.print(rating);
  
  display.display();
  
  // Log to serial
  Serial.print("T: "); Serial.print(temp, 1);
  Serial.print(" C | H: "); Serial.print(hum, 1);
  Serial.print(" % | Gas: "); Serial.print(rawGas);
  Serial.print(" | AQI: "); Serial.println(rating);
  
  delay(1000); // 1Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MQ-2 Sensor**, **DHT22**, **SSD1306 OLED**, and **Buzzer** onto the canvas.
2. Wire MQ-2 to **GPIO34**, DHT22 to **GPIO4**, Buzzer to **GPIO15**, and OLED to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Adjust MQ-2 gas levels. Slide raw values above 1800. Watch the OLED switch to "DANGER!" and the buzzer sound.

## Expected Output
Serial Monitor:
```
T: 24.5 C | H: 45.2 % | Gas: 450 | AQI: SAFE
T: 24.5 C | H: 45.2 % | Gas: 1250 | AQI: WARNING
T: 24.5 C | H: 45.2 % | Gas: 2100 | AQI: DANGER!
```

OLED Display HUD (normal):
```
AIR MONITOR STATION
───────────────────
Temp: 24.5C  Hum: 45%
Gas density: 450
AQI: SAFE
```

## Expected Canvas Behavior
* The OLED widget displays the live temperature, humidity, gas value, and safety status.
* Sliding the gas density slider above 1000 changes the status to "WARNING".
* Sliding the density above 1800 changes the status to "DANGER!" and sounds the buzzer widget.

## Code Walkthrough
| Line | Check / Math |
| --- | --- |
| `rawGas >= GAS_DANGER_LIMIT` | Evaluates if the gas density has crossed the safety threshold. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Activates the buzzer if danger levels are reached. |
| `display.drawFastHLine(...)` | Draws a horizontal dividing line on the OLED buffer to partition layout sections. |

## Hardware & Safety Concept: Multi-Sensor Safety nodes
Safety instrumentation panels combine multiple variables (thermal data and toxic gas densities) to create robust hazard assessments. If gas levels rise, it indicates a leak. If temperature rises as well, it indicates a fire or thermal reaction, triggering higher priority alarms (such as automatic sprinkler or ventilating fans).

## Try This! (Challenges)
1. **Dynamic Flashing Alarm**: Flash the OLED screen backlight inverted (Project 107) when in the DANGER state.
2. **Ventilation Fan relay**: Add a relay on GPIO 13 that turns on a ventilation fan if gas levels exceed 1000.
3. **Sensor pre-heat delay**: Display a countdown timer "Sensor Preheating..." on the OLED for the first 30 seconds of boot.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Gas readings are static or extremely low | MQ-2 heater cold or sensor not powered from 5V | The MQ-2 requires a 5V power source to pre-heat the internal sensing element; wait 60 seconds after powering on |
| OLED displays static or freezes | I2C pull-up resistor missing | Ensure SDA and SCL lines are connected; check I2C address |
| Buzzer does not sound during danger | Incorrect pin mapping | Confirm the active buzzer is connected to GPIO 15 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [43 - ESP32 MQ-2 Gas Sensor Level Print](../beginner/43-esp32-mq2-gas-sensor-level-print.md)
- [98 - ESP32 Gas Leakage Alarm Node](../intermediate/98-esp32-gas-leakage-alarm-node.md)
- [139 - ESP32 Multi-Sensor Environmental HUD](139-esp32-multi-sensor-environmental-hud.md)
