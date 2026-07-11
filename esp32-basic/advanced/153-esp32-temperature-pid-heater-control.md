# 153 - ESP32 Temperature PID Heater Control

Build a slow time-proportioning PID heating controller that reads temperature from a DHT22 sensor, compares it against a target temperature (set by a potentiometer), and modulates a relay ON/OFF time fraction over a 5-second cycle window, displaying status on an LCD.

## Goal
Learn how to implement a time-proportioning PID control loop (slow PWM) for high-inertia thermal systems, avoiding relay wear while maintaining target temperatures.

## What You Will Build
A DHT22 is connected to GPIO 4. A potentiometer on GPIO 34 sets the target temperature (20.0 °C to 40.0 °C). A relay on GPIO 13 controls a heater. A 16x2 LCD displays temperatures and duty cycles. The ESP32 runs a PID loop every 1 second, updating the relay's ON-time fraction of a 5-second cycle window (e.g. 40% duty = relay ON for 2 seconds, OFF for 3 seconds).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Temperature input |
| DHT22 Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| Relay Module | IN (Signal) | GPIO13 | Orange | Heater control |
| Relay Module | VCC / GND | 5V / GND | Red / Black | Power rails |
| Potentiometer | Wiper | GPIO34 | Yellow | Target temp input |
| Potentiometer | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |

> **Wiring tip:** Share the I2C bus pins. Connect the relay signal to GPIO 13. Ensure all grounds are tied together.

## Code
```cpp
// Temperature PID Heater Control (DHT22 + Slow PWM Relay)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

const int DHT_PIN = 4;
const int RELAY_PIN = 13;
const int POT_PIN = 34;

#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// PID Constants
const float KP = 25.0; // High gain since Celsius changes slowly
const float KI = 0.5;
const float KD = 10.0;

float targetTemp = 25.0;
float currentTemp = 20.0;
float error = 0.0;
float lastError = 0.0;
float integral = 0.0;
float derivative = 0.0;

// Slow time-proportioning PWM parameters
const unsigned long WINDOW_SIZE_MS = 5000; // 5-second cycle window
unsigned long windowStartTime = 0;
float dutyPercent = 0.0; // PID output (0% to 100%)

unsigned long lastPIDTime = 0;

void setup() {
  Serial.begin(115200);
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);
  
  dht.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Thermal PID Ready");
  
  windowStartTime = millis();
  lastPIDTime = millis();
  
  delay(1500);
  lcd.clear();
}

void loop() {
  unsigned long now = millis();
  
  // 1. Run PID Loop every 1 second
  if (now - lastPIDTime >= 1000) {
    float dt = (now - lastPIDTime) / 1000.0;
    lastPIDTime = now;
    
    // Read current temperature
    float tempRead = dht.readTemperature();
    if (!isnan(tempRead)) {
      currentTemp = tempRead;
    }
    
    // Read target temperature from pot (20C to 40C)
    int potRaw = analogRead(POT_PIN);
    targetTemp = 20.0 + (potRaw * 20.0 / 4095.0);
    
    // Calculate Error (Target - Current)
    error = targetTemp - currentTemp;
    
    // Calculate Integral with windup clamping
    if (error > -5.0 && error < 5.0) { // Only integrate close to target
      integral += error * dt;
    } else {
      integral = 0.0;
    }
    integral = constrain(integral, -10, 10);
    
    // Calculate Derivative
    derivative = (error - lastError) / dt;
    lastError = error;
    
    // Calculate PID output (duty cycle percentage: 0% to 100%)
    float output = (KP * error) + (KI * integral) + (KD * derivative);
    dutyPercent = constrain(output, 0.0, 100.0);
    
    // Update LCD
    lcd.setCursor(0, 0);
    lcd.print("T:"); lcd.print(currentTemp, 1);
    lcd.print("C  Set:"); lcd.print(targetTemp, 1); lcd.print("C ");
    
    lcd.setCursor(0, 1);
    lcd.print("Heater: "); lcd.print(dutyPercent, 0); lcd.print("%    ");
    
    Serial.print("Target: "); Serial.print(targetTemp, 1);
    Serial.print(" | Temp: "); Serial.print(currentTemp, 1);
    Serial.print(" | Error: "); Serial.print(error, 1);
    Serial.print(" | Duty: "); Serial.print(dutyPercent, 0); Serial.println("%");
  }
  
  // 2. Drive Relay using time-proportioning (Slow PWM)
  unsigned long timeInWindow = now - windowStartTime;
  
  // Reset window start time
  if (timeInWindow >= WINDOW_SIZE_MS) {
    windowStartTime = now;
    timeInWindow = 0;
  }
  
  // Calculate ON duration in milliseconds
  unsigned long onDuration = (dutyPercent / 100.0) * WINDOW_SIZE_MS;
  
  if (timeInWindow < onDuration) {
    digitalWrite(RELAY_PIN, HIGH); // Turn heater ON
  } else {
    digitalWrite(RELAY_PIN, LOW);  // Turn heater OFF
  }
  
  delay(20); // Fast loop for relay switching precision
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, **Relay**, **Potentiometer**, and **16x2 I2C LCD** onto the canvas.
2. Wire components: DHT22 to **GPIO4**, Relay to **GPIO13**, Potentiometer to **GPIO34**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the potentiometer to set a target temperature (e.g. 30 °C). Slide the DHT22 temperature to 20 °C. Watch the relay turn ON.
5. Raise the DHT22 temperature to 29.5 °C. Watch the relay begin to pulse ON and OFF every 5 seconds (time-proportioning).

## Expected Output
Serial Monitor:
```
Target: 30.0 | Temp: 20.0 | Error: 10.0 | Duty: 100%
Target: 30.0 | Temp: 29.5 | Error: 0.5 | Duty: 12%
Target: 30.0 | Temp: 30.2 | Error: -0.2 | Duty: 0%
```

LCD Display:
```
T:29.5C  Set:30.0C
Heater: 12%
```

## Expected Canvas Behavior
* If the temperature is far below the target, the relay widget stays ON (green).
* As the temperature gets closer to the target, the relay widget pulses green (ON) and grey (OFF) over a 5-second cycle window.
* If the temperature exceeds the target, the relay stays OFF (grey).

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `timeInWindow >= WINDOW_SIZE_MS` | Resets the 5-second time frame loop. |
| `onDuration = (dutyPercent / 100.0) * ...` | Calculates how many milliseconds the relay must stay closed during the window. |
| `timeInWindow < onDuration` | Drives the relay HIGH for the calculated duration, and LOW for the remaining time. |

## Hardware & Safety Concept: Time-Proportioning Controllers and Relay Wear
Relay modules contain mechanical coils and contacts that wear out if switched too frequently. Standard high-frequency PWM (e.g. 1 kHz) would burn out a relay in minutes. Heating systems use **time-proportioning control (slow PWM)**, using a cycle window of 5 to 30 seconds. This allows precise proportional heating without overloading the mechanical contacts.

## Try This! (Challenges)
1. **Sensors Fault Shutdown safety**: Add a cutoff that shuts off the relay and flashes "Sensor Error" if the DHT22 fails.
2. **Dynamic Window Adjuster**: Modify the code to shorten the window size if a solid-state relay (SSR) is used.
3. **Integral anti-windup clamping**: Implement stricter clamping to prevent temperature overshoot.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay clicks continuously and rapidly | PWM logic error | Ensure the relay is driven using the slow time-proportioning window, not standard analog/PWM writes |
| Temperature overshoots target | `KI` is set too high | Decrease `KI` or restrict integration to when error is < 2 °C |
| Temperature never reaches target | Heater too weak or `KP` too low | Increase the proportional gain `KP` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [105 - ESP32 Thermostat Controller](105-esp32-thermostat-controller.md)
- [151 - ESP32 DC Motor PID Speed Controller](151-esp32-dc-motor-pid-speed-controller.md)
- [73 - ESP32 DHT22 Temperature LCD HUD](../intermediate/73-esp32-dht22-temperature-lcd-hud.md)
