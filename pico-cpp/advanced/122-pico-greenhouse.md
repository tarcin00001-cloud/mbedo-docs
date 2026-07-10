# 122 - Pico Greenhouse

Build an automated greenhouse climate controller that waters soil based on moisture and runs ventilation fans based on temperature.

## Goal
Learn how to read multiple analog and digital sensors (Soil Moisture, DHT22) to actuate separate output systems (irrigation relay, DC motor fan) while logging status metrics on an LCD screen.

## What You Will Build
A smart greenhouse climate controller:
- **DHT22 Sensor (GP12)**: Measures ambient temperature.
- **Soil Moisture Sensor (GP26)**: Monitors soil dryness.
- **Relay Module (GP10)**: Actuates a 5V solenoid water valve if the soil is dry.
- **DC Motor Fan (GP13-15 via L298N)**: Speed scales to full if temperature exceeds 30.0°C.
- **16x2 I2C LCD (GP4, GP5)**: Displays soil moisture, temperature, and system actions.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| Soil Moisture Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Relay Module | `relay` | Yes | Yes |
| DC Motor | `dc_motor` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DHT22 | SDA | GP12 | Climate sensor (requires 10k pull-up) |
| Soil Sensor | AO | GP26 | Moisture level signal |
| Relay Module | IN | GP10 | Solenoid valve control |
| L298N Control | ENA | GP13 | Fan speed control |
| L298N Control | IN1 / IN2 | GP14 / GP15 | Fan direction control |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <LiquidCrystal_I2C.h>

const int DHT_PIN   = 12;
const int SOIL_PIN  = 26;
const int VALVE_PIN = 10;
const int FAN_ENA   = 13;
const int FAN_IN1   = 14;
const int FAN_IN2   = 15;

#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Thresholds
const int SOIL_DRY_LIMIT  = 2800; // High raw reading = dry soil
const float TEMP_HOT_LIMIT = 30.0; // Heat limit

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  dht.begin();
  lcd.init();
  lcd.backlight();

  pinMode(VALVE_PIN, OUTPUT);
  pinMode(FAN_ENA, OUTPUT);
  pinMode(FAN_IN1, OUTPUT);
  pinMode(FAN_IN2, OUTPUT);

  // Set fan forward direction
  digitalWrite(FAN_IN1, HIGH);
  digitalWrite(FAN_IN2, LOW);

  digitalWrite(VALVE_PIN, LOW);
  analogWrite(FAN_ENA, 0); // Start fan stopped
}

void loop() {
  float temp = dht.readTemperature();
  int moisture = analogRead(SOIL_PIN);

  if (!isnan(temp)) {
    lcd.clear();

    // 1. Irrigation Control
    if (moisture > SOIL_DRY_LIMIT) {
      digitalWrite(VALVE_PIN, HIGH); // Open water valve
      lcd.setCursor(0, 0);
      lcd.print("Water: ON ");
    } else {
      digitalWrite(VALVE_PIN, LOW);  // Close water valve
      lcd.setCursor(0, 0);
      lcd.print("Water: OFF");
    }

    // Print temperature on Row 0 right side
    lcd.print(" T:");
    lcd.print((int)temp);
    lcd.print("C");

    // 2. Ventilation Control
    lcd.setCursor(0, 1);
    if (temp > TEMP_HOT_LIMIT) {
      analogWrite(FAN_ENA, 255); // Full power cooling
      lcd.print("Fan: FULL COOL  ");
    } else if (temp > 25.0) {
      analogWrite(FAN_ENA, 128); // Low power circulation
      lcd.print("Fan: LOW CIRC   ");
    } else {
      analogWrite(FAN_ENA, 0);   // Stop fan
      lcd.print("Fan: STOPPED    ");
    }
  }

  delay(2000); // DHT22 rate limit
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **Soil Sensor** (potentiometer), **Relay**, **L298N**, **DC Motor**, and **I2C LCD** onto the canvas.
2. Connect DHT22 to **GP12**, Soil Sensor to **GP26**, Relay to **GP10**, L298N to **GP13-15**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the temperature and soil moisture sliders and watch the water valve, fan, and LCD update.

## Expected Output

Terminal:
```
Simulation active. Automated greenhouse controller online.
```

## Expected Canvas Behavior
* Hot & Dry (Temp 32°C, Moisture 3000): LCD reads `Water: ON  T:32C` / `Fan: FULL COOL`. Relay is ON, DC Motor runs at full speed.
* Cold & Wet (Temp 22°C, Moisture 1000): LCD reads `Water: OFF T:22C` / `Fan: STOPPED`. Relay is OFF, DC Motor is stopped.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `moisture > SOIL_DRY_LIMIT` | Evaluates if the soil has dried out to open the irrigation solenoid valve. |
| `analogWrite(FAN_ENA, 128)` | Runs the circulation fan at half power when temperature is moderate. |

## Hardware & Safety Concept: Crop Care Automation
Smart greenhouses automate microclimate variables to maximize crop yields. If humidity levels drop too low or temperatures rise too high, plants suffer thermal stress. Automated irrigation and ventilation loops keep these variables within optimal growth zones. Real greenhouse nodes also monitor light levels to control motorized shade screens.

## Try This! (Challenges)
1. **Freeze Alert Chime**: Connect a buzzer on GP11 and sound a warning beep if the temperature drops below 5.0°C to protect crops from frost.
2. **Light Control**: Add an LDR sensor on GP27 and switch a grow light relay on GP18 if it is too dark.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD flashes on and off constantly | Too many clears in loop | Make sure `lcd.clear()` and screen prints are located inside the slow 2-second loop cycle. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [100 - Pico Smart Fan](../intermediate/100-pico-smart-fan.md)
- [110 - Pico Irrigation Station](../intermediate/110-pico-irrigation-station.md)
