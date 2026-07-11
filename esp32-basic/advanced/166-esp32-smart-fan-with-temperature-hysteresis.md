# 166 - ESP32 Smart Fan with Temperature Hysteresis

Build a smart cooling fan controller that monitors room temperature using a DHT22 sensor, drives a cooling fan using a relay or L298N driver, and implements a hysteresis control loop to prevent rapid fan switching (chatter) near the threshold, displaying status on an LCD.

## Goal
Learn how to implement dual-threshold hysteresis control logic, manage thermal inertia parameters, and prevent actuator wear caused by noise.

## What You Will Build
A DHT22 sensor is connected to GPIO 4. An L298N driver controls a DC cooling fan motor (IN1: 18, IN2: 19, ENA: 5). A 16x2 I2C LCD displays temperatures and fan status. The fan turns ON when temperature rises above 30.0 °C, and stays ON until temperature drops below 27.0 °C.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motor (Cooling Fan) | `dc_motor` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Temperature input |
| L298N Module | IN1 / IN2 / ENA | GPIO18 / GPIO19 / GPIO5 | Yellow/Green/Orange | Fan motor control |
| L298N Module | GND | GND | Black | Shared logic ground |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus pins. Power the fan motor from a dedicated battery pack or external supply connected to the L298N power terminals.

## Code
```cpp
// Smart Fan with Temperature Hysteresis (DHT22 + L298N + LCD)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

const int DHT_PIN = 4;
const int IN1 = 18;
const int IN2 = 19;
const int ENA = 5;

#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Hysteresis Limits
// Fan starts above this limit
const float TEMP_HIGH_LIMIT = 30.0; 
// Fan stops below this limit
const float TEMP_LOW_LIMIT = 27.0;  

bool fanActive = false;

void turnFanOn() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(ENA, HIGH);
  fanActive = true;
  Serial.println(">> FAN STARTED <<");
}

void turnFanOff() {
  digitalWrite(ENA, LOW);
  fanActive = false;
  Serial.println(">> FAN STOPPED <<");
}

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(ENA, OUTPUT);
  
  turnFanOff(); // Start stopped
  
  dht.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Fan Station");
  lcd.setCursor(0, 1);
  lcd.print("Hysteresis Loop");
  
  delay(1500);
  lcd.clear();
}

void loop() {
  float temp = dht.readTemperature();
  
  if (!isnan(temp)) {
    // Hysteresis Loop logic
    // 1. High Temperature threshold -> turn fan ON
    if (temp >= TEMP_HIGH_LIMIT && !fanActive) {
      turnFanOn();
    }
    // 2. Low Temperature threshold -> turn fan OFF
    else if (temp <= TEMP_LOW_LIMIT && fanActive) {
      turnFanOff();
    }
    
    // Update LCD Screen
    lcd.setCursor(0, 0);
    lcd.print("Temp: ");
    lcd.print(temp, 1);
    lcd.print((char)223); // Degree symbol
    lcd.print("C    ");
    
    lcd.setCursor(0, 1);
    lcd.print("Fan Status: ");
    lcd.print(fanActive ? "ON " : "OFF");
    
    Serial.print("Temp: "); Serial.print(temp, 1);
    Serial.print(" C | Fan: "); Serial.println(fanActive ? "ON" : "OFF");
  } else {
    Serial.println("Sensor Read Error!");
    lcd.setCursor(0, 0);
    lcd.print("DHT Sensor Error");
    turnFanOff(); // Safe shutdown on error
  }
  
  delay(1500); // 1.5-second sampling rate (matches DHT22 limits)
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, **L298N**, **DC Motor**, and **16x2 I2C LCD** onto the canvas.
2. Wire components: DHT22 to **GPIO4**, motor pins to **18, 19, 5**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the DHT22 temperature to 32 °C. Watch the fan motor turn ON.
5. Slide the temperature to 28.5 °C. Watch the fan motor stay ON (hysteresis).
6. Slide the temperature to 26 °C. Watch the fan motor turn OFF.

## Expected Output
Serial Monitor:
```
Temp: 26.5 C | Fan: OFF
Temp: 30.5 C | Fan: OFF
>> FAN STARTED <<
Temp: 28.5 C | Fan: ON   (Hysteresis: stays ON!)
Temp: 26.5 C | Fan: ON
>> FAN STOPPED <<
```

LCD Display:
```
Temp: 28.5°C
Fan Status: ON
```

## Expected Canvas Behavior
* At boot, the LCD initializes.
* Raising the temperature slider past 30.0 °C starts the DC motor widget spinning clockwise.
* Lowering the temperature to 28.0 °C does not stop the motor.
* The motor stops spinning only when the temperature drops below 27.0 °C.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `temp >= TEMP_HIGH_LIMIT` | Starts the cooling fan once the high temperature limit is exceeded. |
| `temp <= TEMP_LOW_LIMIT` | Shuts off the fan once the temperature drops below the lower limit. |
| `turnFanOff() (on error)` | Safe shutdown state if the sensor fails or is disconnected. |

## Hardware & Safety Concept: Actuator Chatter and Hysteresis
If a cooling fan is controlled by a single threshold (e.g. ON above 30.0 °C, OFF below 30.0 °C), any noise in the temperature readings when hovering near 30.0 °C (e.g. jumping between 29.9 °C and 30.1 °C) causes the fan to switch ON and OFF rapidly. This **actuator chatter** causes excessive noise and electrical wear, quickly burning out motors and relays. Implementing **hysteresis** (separate high and low thresholds) creates a deadband that prevents chatter.

## Try This! (Challenges)
1. **Dynamic Hysteresis dial**: Add a potentiometer (GPIO 34) to allow adjusting the high temperature threshold from 25 °C to 35 °C.
2. **Proportional Speed override**: If the temperature exceeds 35 °C, override hysteresis and drive the fan at full speed.
3. **SD Card Data Logging**: Integrate an SD card (Project 137) to log temperature and fan status.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Fan turns ON and OFF rapidly at 30 °C | Hysteresis logic missing or threshold values identical | Check that `TEMP_HIGH_LIMIT` and `TEMP_LOW_LIMIT` are set to different values in the code |
| LCD displays "DHT Sensor Error" | Sensor connection issue | Verify DHT22 data pin is connected to GPIO 4 |
| Fan motor does not spin | External power missing | Ensure L298N power terminals are connected to a suitable supply |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [100 - ESP32 Automatic Smart Fan](../intermediate/100-esp32-automatic-smart-fan.md)
- [153 - ESP32 Temperature PID Heater Control](153-esp32-temperature-pid-heater-control.md)
- [73 - ESP32 DHT22 Temperature LCD HUD](../intermediate/73-esp32-dht22-temperature-lcd-hud.md)
