# 180 - Automated Greenhouse

Create a smart climate control system for greenhouses using a DHT22, soil moisture and water level sensors, and three relay outputs to regulate irrigation, cooling, and heating.

## Goal
Learn how to parse multiple analog and digital sensors simultaneously, implement threshold-based control rules with dry-run water pump protection, and run a multi-page LCD status dashboard without loops.

## What You Will Build
An automated greenhouse manager. The system monitors ambient temperature, humidity, soil moisture, and reservoir water level. If the soil is dry, it activates the water pump relay (unless the water reservoir is empty, which triggers dry-run protection and an alarm). If the greenhouse gets too hot, it turns on the ventilation fan relay. If it gets too cold, it activates the heater relay. A rotating LCD display shows current status pages.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| Soil Moisture Sensor | `soil_moisture` | Yes | Yes |
| Water Level Sensor | `water_level` | Yes | Yes |
| 3x Relay Modules | `relay` | Yes | Yes |
| I2C LCD (16x2) | `lcd_i2c` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO 12 | Yellow | Temp/humidity input |
| DHT22 Sensor | VCC | 3V3 | Red | Power |
| DHT22 Sensor | GND | GND | Black | Ground |
| Soil Moisture | OUT (analog) | ADC3 | Blue | Soil moisture input (GP29) |
| Water Level | OUT (analog) | ADC0 | Orange | Water level input (GP26) |
| Relay 1 | Signal | GPIO 7 | White | Water Pump control |
| Relay 2 | Signal | GPIO 8 | Purple | Ventilation Fan control |
| Relay 3 | Signal | GPIO 9 | Brown | Heater control |
| I2C LCD (16x2) | SDA | GPIO 17 | Green | SDA0 |
| I2C LCD (16x2) | SCL | GPIO 16 | Yellow | SCL0 |
| Active Buzzer | + | GPIO 14 | Gray | Low reservoir alarm |
| Active Buzzer | - | GND | Black | Ground |
| Warning LED | Anode | GPIO 15 | Red | Fault indicator |
| Warning LED | Cathode | GND | Black | Ground |

> **Wiring tip:** Relays require 5V supply to operate their internal electromagnets, but their control inputs must be driven safely by the ARIES 3.3V logic outputs. Verify your relay modules support 3.3V logic trigger signals.

## Code
```cpp
// Automated Greenhouse - VEGA ARIES v3
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

const int DHT_PIN = 12;
const int SOIL_PIN = ADC3;
const int WATER_PIN = ADC0;
const int PUMP_RELAY = 7;
const int FAN_RELAY = 8;
const int HEAT_RELAY = 9;
const int BUZZER = 14;
const int LED_PIN = 15;

LiquidCrystal_I2C lcd(0x27, 16, 2);
DHT dht(DHT_PIN, DHT22);

unsigned long lastSampleTime = 0;
unsigned long lastPageTime = 0;
int lcdPage = 0; // 0: Temp/Hum, 1: Soil/Water, 2: Relay status

// Telemetry State Variables
float tempVal = 0.0;
float humVal = 0.0;
int soilPercent = 0;
int waterPercent = 0;

bool pumpActive = false;
bool fanActive = false;
bool heatActive = false;
bool lowWaterAlarm = false;

void setup() {
  Serial.begin(115200);
  pinMode(PUMP_RELAY, OUTPUT);
  pinMode(FAN_RELAY, OUTPUT);
  pinMode(HEAT_RELAY, OUTPUT);
  pinMode(BUZZER, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  
  dht.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Greenhouse Init");
  lcd.setCursor(0, 1);
  lcd.print("Starting...");
  
  lastSampleTime = millis();
  lastPageTime = millis();
}

void loop() {
  unsigned long currentTime = millis();
  
  // 1. Read Sensors and run Control Logic every 2 seconds
  if (currentTime - lastSampleTime >= 2000) {
    lastSampleTime = currentTime;
    
    // Read Sensors
    tempVal = dht.readTemperature();
    humVal = dht.readHumidity();
    
    // Analog conversions (assuming standard 0-1023 ranges)
    soilPercent = map(analogRead(SOIL_PIN), 0, 1023, 0, 100);
    waterPercent = map(analogRead(WATER_PIN), 0, 1023, 0, 100);
    
    // Safety check: Dry Run Protection for Water Pump
    if (waterPercent <= 15) {
      lowWaterAlarm = true;
      pumpActive = false; // Force pump off
      digitalWrite(PUMP_RELAY, LOW);
      digitalWrite(BUZZER, HIGH);
      digitalWrite(LED_PIN, HIGH);
    } else {
      lowWaterAlarm = false;
      digitalWrite(BUZZER, LOW);
      digitalWrite(LED_PIN, LOW);
      
      // Auto Watering rule
      if (soilPercent < 45) {
        pumpActive = true;
        digitalWrite(PUMP_RELAY, HIGH);
      } else {
        pumpActive = false;
        digitalWrite(PUMP_RELAY, LOW);
      }
    }
    
    // Auto Cooling (Ventilation Fan) rule
    if (tempVal > 30.0) {
      fanActive = true;
      digitalWrite(FAN_RELAY, HIGH);
    } else {
      fanActive = false;
      digitalWrite(FAN_RELAY, LOW);
    }
    
    // Auto Heating rule
    if (tempVal < 18.0) {
      heatActive = true;
      digitalWrite(HEAT_RELAY, HIGH);
    } else {
      heatActive = false;
      digitalWrite(HEAT_RELAY, LOW);
    }
    
    // Print logs to Serial Monitor
    Serial.print("T:"); Serial.print(tempVal);
    Serial.print("C | H:"); Serial.print(humVal);
    Serial.print("% | Soil:"); Serial.print(soilPercent);
    Serial.print("% | Water:"); Serial.print(waterPercent);
    Serial.print("% | Pump:"); Serial.print(pumpActive ? "ON" : "OFF");
    Serial.print(" Fan:"); Serial.print(fanActive ? "ON" : "OFF");
    Serial.print(" Heat:"); Serial.println(heatActive ? "ON" : "OFF");
  }
  
  // 2. Rotate LCD Display page every 3 seconds
  if (currentTime - lastPageTime >= 3000) {
    lastPageTime = currentTime;
    lcd.clear();
    
    if (lcdPage == 0) {
      lcd.setCursor(0, 0);
      lcd.print("Temp: ");
      lcd.print(tempVal, 1);
      lcd.print(" C");
      lcd.setCursor(0, 1);
      lcd.print("Humidity: ");
      lcd.print(humVal, 0);
      lcd.print(" %");
      lcdPage = 1;
    } 
    else if (lcdPage == 1) {
      lcd.setCursor(0, 0);
      lcd.print("Soil Moist: ");
      lcd.print(soilPercent);
      lcd.print("%");
      lcd.setCursor(0, 1);
      if (lowWaterAlarm) {
        lcd.print("LOW WATER LEVEL!");
      } else {
        lcd.print("Reservoir: ");
        lcd.print(waterPercent);
        lcd.print("%");
      }
      lcdPage = 2;
    } 
    else if (lcdPage == 2) {
      lcd.setCursor(0, 0);
      lcd.print("Pump:");
      lcd.print(pumpActive ? "ON" : "OFF");
      lcd.print(" Fan:");
      lcd.print(fanActive ? "ON" : "OFF");
      lcd.setCursor(0, 1);
      lcd.print("Heater:");
      lcd.print(heatActive ? "ON" : "OFF");
      lcdPage = 0;
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DHT22**, **Soil Moisture**, **Water Level**, **three Relays**, **I2C LCD**, **Buzzer**, and **LED** onto the canvas.
2. Complete the wiring matching the table.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the dropdown.
5. Click **Run**.
6. Alter the water level and soil moisture sliders on their respective widgets to observe the relays turning on/off and warning behaviors.

## Expected Output
Serial Monitor:
```
T:24.50C | H:48.00% | Soil:38% | Water:85% | Pump:ON Fan:OFF Heat:OFF
T:32.40C | H:42.00% | Soil:55% | Water:80% | Pump:OFF Fan:ON Heat:OFF
T:22.10C | H:50.00% | Soil:30% | Water:8% | Pump:OFF Fan:OFF Heat:OFF (LOW WATER ALERT)
```

## Expected Canvas Behavior
* The LCD page alternates every 3 seconds showing different telemetry views.
* If you set soil moisture slider < 45% and water level > 15%, the Pump Relay (GPIO 7) turns on (lights green).
* If water level slider drops to 15% or less, the buzzer turns on, the red LED turns on, the Pump Relay turns OFF immediately, and page 2 of the LCD flashes "LOW WATER LEVEL!".
* Increasing temperature widget > 30°C triggers the Fan Relay (GPIO 8); decreasing < 18°C triggers the Heater Relay (GPIO 9).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `waterPercent <= 15` | Implements safety check for water reservoir level before turning pump relay on. |
| `digitalWrite(PUMP_RELAY, HIGH)` | Drives GPIO 7 high to switch the irrigation pump control relay on. |
| `tempVal > 30.0` | Activates ventilation fan relay to lower temperature inside greenhouse. |
| `currentTime - lastPageTime >= 3000` | Cycles through three layout views of LCD data using clock states without delays. |
| `lcdPage = 1` | Sets the step index for the LCD page rotation state machine. |

## Hardware & Safety Concept
* **Dry Run Damage Prevention**: Submersible water pumps rely on surrounding water for cooling and lubrication. Running a pump dry causes rapid friction overheating and motor failure. The software interlock enforces pump safety.
* **Galvanic Isolation**: High-voltage water pumps or space heaters operate on AC mains voltages. Relays isolate the safe 3.3V logic ARIES board from hazardous voltages using magnetic induction coils.

## Try This! (Challenges)
1. **Humidity Ventilation Rule**: If ambient humidity exceeds 80%, activate the Fan Relay even if the temperature is normal to reduce mold risks.
2. **Irrigation Timeout**: Add a timer check that prevents the water pump from running continuously for more than 15 seconds, avoiding greenhouse flooding in case of sensor failures.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Water pump does not turn off when soil is wet | Relay wired to Normally Closed (NC) pin | Wire the water pump load through the Normally Open (NO) and Common (COM) pins of the relay. |
| LCD does not cycle pages | Timer state variable conflict | Ensure `lastPageTime` is updated inside the LCD page cycle `if` statement block. |
| Soil moisture reads constant | Sensor not fully connected to analog pin | Verify that the sensor analog signal line is connected to ADC3. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [72 - DHT22 Temperature Alarm](../intermediate/72-dht22-temperature-alarm.md)
- [163 - Water Quality Station](../advanced/163-water-quality-station.md)
- [182 - Smart Parking Barrier](182-smart-parking-barrier.md)
