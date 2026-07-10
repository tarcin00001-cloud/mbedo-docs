# 195 - Pico Greenhouse Logger OLED

Build an automated crop climate station that controls irrigation valves and ventilation fans, updates an OLED screen, and logs records over Bluetooth.

## Goal
Learn how to monitor multiple analog inputs (soil moisture, reservoir water level), read climate sensors (DHT22), control dual high-load relays, design graphic OLED dashboards, and stream wireless telemetry logs.

## What You Will Build
An automated greenhouse manager:
- **DHT22 Climate Sensor (GP12)**: Measures temperature and humidity.
- **Soil Moisture Sensor (GP27)**: Measures soil moisture.
- **Reservoir Level Sensor (GP26)**: Monitors irrigation reservoir water levels.
- **Irrigation Valve Relay 1 (GP10)**: Toggles water flow.
- **Ventilation Fan Relay 2 (GP11)**: Controls cooling fan speed.
- **SSD1306 OLED (GP4, GP5)**: Displays soil moisture, temperature, and valve/fan statuses.
- **HC-05 Bluetooth Module (GP0, GP1)**: Streams telemetry logs wirelessly.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Climate Sensor | `dht22` | Yes | Yes |
| Soil Moisture Sensor | `potentiometer` | Yes (represented by potentiometer) | Yes |
| Water Level Sensor | `potentiometer` | Yes (represented by potentiometer) | Yes (reservoir monitor) |
| Relay Modules | `relay` | Yes (two relays) | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DHT22 | SDA | GP12 | Climate sensor (requires 10k pull-up) |
| Soil Sensor | AO | GP27 | Analog soil moisture input |
| Water Sensor | AO | GP26 | Analog reservoir level input |
| Relay 1 | IN | GP10 | Irrigation valve solenoid |
| Relay 2 | IN | GP11 | Ventilation fan motor |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int DHT_PIN    = 12;
const int SOIL_PIN   = 27;
const int WATER_PIN  = 26;
const int VALVE_PIN  = 10;
const int FAN_PIN    = 11;

#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

const int DRY_LIMIT  = 2800; // Dry soil threshold (raw ADC)
const int LOW_WATER  = 800;  // Empty reservoir threshold (raw ADC)
const float HOT_TEMP = 30.0; // Cooling fan temperature threshold

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART
  dht.begin();

  pinMode(VALVE_PIN, OUTPUT);
  pinMode(FAN_PIN, OUTPUT);

  digitalWrite(VALVE_PIN, LOW); // Close valve
  digitalWrite(FAN_PIN, LOW);   // Turn fan OFF

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  // Print CSV Header over Bluetooth
  Serial1.println("Temp_C,Hum_pct,Soil_idx,Water_idx,Valve_ON,Fan_ON");
  delay(1000);
}

void loop() {
  float temp  = dht.readTemperature();
  float humid = dht.readHumidity();
  int soil    = analogRead(SOIL_PIN);
  int water   = analogRead(WATER_PIN);

  bool valveActive = false;
  bool fanActive   = false;

  // 1. Irrigation Logic: Water dry soil ONLY if reservoir has water
  if (soil > DRY_LIMIT && water > LOW_WATER) {
    digitalWrite(VALVE_PIN, HIGH);
    valveActive = true;
  } else {
    digitalWrite(VALVE_PIN, LOW);
  }

  // 2. Cooling Ventilation Logic: Turn fan ON if temp exceeds limit
  if (!isnan(temp) && temp > HOT_TEMP) {
    digitalWrite(FAN_PIN, HIGH);
    fanActive = true;
  } else {
    digitalWrite(FAN_PIN, LOW);
  }

  // 3. Update OLED Screen
  if (!isnan(temp)) {
    display.clearDisplay();

    // Draw header bar
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(22, 3);
    display.print("GREENHOUSE HUD");

    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);
    
    // Row 1: Temp & Soil
    display.setCursor(8, 20);
    display.print("Temp: ");
    display.print(temp, 1);
    display.print("C | S: ");
    display.print(soil);

    // Row 2: Water Level
    display.setCursor(8, 34);
    display.print("Water Level: ");
    display.print(water);

    // Row 3: Actuator Statuses
    display.setCursor(8, 48);
    display.print("Valve:");
    display.print(valveActive ? "ON " : "OFF");
    display.print(" | Fan:");
    display.print(fanActive ? "ON" : "OFF");

    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();
  }

  // 4. Stream telemetry CSV logs over Bluetooth
  if (!isnan(temp)) {
    Serial1.print(temp, 2);
    Serial1.print(",");
    Serial1.print(humid, 2);
    Serial1.print(",");
    Serial1.print(soil);
    Serial1.print(",");
    Serial1.print(water);
    Serial1.print(",");
    Serial1.print(valveActive ? 1 : 0);
    Serial1.print(",");
    Serial1.println(fanActive ? 1 : 0);
  }

  delay(3000); // Update once every 3 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **two potentiometers** (soil/water), **two Relays**, **SSD1306 OLED**, and **HC-05** onto the canvas.
2. Connect sensors, relays, screen, and Bluetooth.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the soil moisture slider to dry and the water reservoir slider to full, and verify that the irrigation valve relay activates.

## Expected Output

Terminal (Bluetooth):
```
Temp_C,Hum_pct,Soil_idx,Water_idx,Valve_ON,Fan_ON
24.50,50.20,1200,3000,0,0
24.60,50.30,3200,3000,1,0
31.50,48.20,3200,3000,1,1
```

## Expected Canvas Behavior
* Normal state (Moist soil, Cool air): Valve and Fan Relays are OFF.
* Soil Dry (> 2800) + Reservoir Full (> 800): Relay 1 turns ON (Valve opens). OLED reads `Valve:ON`.
* Air Hot (> 30°C): Relay 2 turns ON (Fan runs). OLED reads `Fan:ON`.
* Reservoir Empty (< 800): Relay 1 turns OFF immediately (failsafe protection).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `soil > DRY_LIMIT && water > LOW_WATER` | Safeguard rule that opens the water valve only when the soil is dry AND the reservoir has enough water to prevent running the pump dry. |

## Hardware & Safety Concept: Dry-Run Pump Protection
Running water pumps when the reservoir is empty (dry running) causes rapid friction wear and overheating, destroying the pump motor. Smart greenhouse controllers add water level sensors to the reservoir and override irrigation commands to shut down the pump immediately if water levels drop below safety limits.

## Try This! (Challenges)
1. **Critical Water Alarm**: Sound a buzzer on GP14 if the water reservoir runs dry.
2. **Display Sleep**: Turn OFF the OLED screen automatically if no alarms are active to save energy.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Irrigation valve opens even when reservoir is empty | Logic check failure | Ensure the code uses logical AND (`&&`) to verify both soil moisture and water level before starting the pump. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [122 - Pico Greenhouse](../advanced/122-pico-greenhouse.md)
- [130 - Pico Soil Irrigator](../advanced/130-pico-soil-irrigator.md)
- [178 - Pico Greenhouse Datalogger](178-pico-greenhouse-datalogger.md)
