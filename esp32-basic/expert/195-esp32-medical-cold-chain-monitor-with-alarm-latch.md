# 195 - ESP32 Medical Cold Chain Monitor with Alarm Latch

Build an industrial medical cold chain safety monitor on the ESP32 that reads a DS18B20 temperature sensor, controls a cooling relay, and triggers a latched buzzer alarm if the temperature drifts outside the 2.0 °C to 8.0 °C range, requiring a button press to clear the alarm.

## Goal
Learn how to implement alarm latch logic for critical storage applications, read OneWire sensors, and build safety override loops.

## What You Will Build
A DS18B20 digital temperature sensor is on GPIO 4. A cooling compressor relay is on GPIO 13, an active buzzer on GPIO 15, a push button on GPIO 12, and a 16x2 LCD on I2C. If the temperature exceeds 8.0 °C or drops below 2.0 °C, the buzzer turns ON. The alarm stays active (latched) even if the temperature returns to normal, until the button is pressed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DS18B20 Waterproof Temp Sensor | `ds18b20` | Yes | Yes |
| Relay Module (Cooler) | `relay` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| 4.7 kΩ Resistor (OneWire pull-up) | `resistor` | No | Yes |
| 10 kΩ Resistor (for button) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS18B20 Sensor | DQ (Data) | GPIO4 | Green | Shared OneWire bus |
| 4.7 kΩ Resistor | Leg 1 / Leg 2 | GPIO4 / 3V3 | White | Bus pull-up resistor |
| DS18B20 Sensor | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |
| Relay Module | IN (Signal) | GPIO13 | Orange | Cooler compressor switch |
| Push Button | Pin 1 / Pin 2 | GPIO12 / 3V3 | Green / Red | Alarm Reset (active-HIGH) |
| Push Button | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm siren |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |

> **Wiring tip:** The DS18B20 requires a 4.7 kΩ pull-up resistor between the data pin (GPIO 4) and 3.3V. Connect the button with an external 10 kΩ pull-down resistor to GPIO 12.

## Code
```cpp
// Medical Cold Chain Monitor with Alarm Latch (DS18B20 + Buzzer + Relay + Button)
#include <OneWire.h>
#include <DallasTemperature.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int ONE_WIRE_BUS = 4;
const int RELAY_PIN = 13;
const int BUTTON_PIN = 12;
const int BUZZER_PIN = 15;

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Medical storage safety range (e.g. Vaccine cold storage)
const float TEMP_SAFE_MIN = 2.0;
const float TEMP_SAFE_MAX = 8.0;

// Alarm Latch flag
bool alarmLatched = false;
float peakExcursionTemp = 5.0; // Track the highest/lowest temp reached during alarm

unsigned long lastSampleTime = 0;
const unsigned long SAMPLE_INTERVAL_MS = 2000; // Sample once every 2 seconds

void setup() {
  Serial.begin(115200);
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT); // External pull-down wired
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  
  sensors.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Cold Chain Mon");
  lcd.setCursor(0, 1);
  lcd.print("Active...");
  
  delay(1500);
  lcd.clear();
}

void loop() {
  unsigned long now = millis();
  
  // Read reset button
  bool resetPressed = (digitalRead(BUTTON_PIN) == HIGH);
  
  // 1. Read temperature every 2 seconds
  if (now - lastSampleTime >= SAMPLE_INTERVAL_MS) {
    sensors.requestTemperatures();
    float temp = sensors.getTempCByIndex(0);
    
    if (temp != DEVICE_DISCONNECTED_C) {
      // 2. Evaluate Hysteresis Compressor Control
      // Turn cooler ON if temperature is high (> 6.0C)
      if (temp > 6.0) {
        digitalWrite(RELAY_PIN, HIGH); // Start cooler
      } else if (temp < 4.0) {
        digitalWrite(RELAY_PIN, LOW);  // Stop cooler
      }
      
      // 3. Evaluate Out-of-bounds Alarm
      bool isOutOfBounds = (temp < TEMP_SAFE_MIN || temp > TEMP_SAFE_MAX);
      
      if (isOutOfBounds) {
        if (!alarmLatched) {
          alarmLatched = true;
          peakExcursionTemp = temp; // Seed peak temp
          Serial.println("!!! ALARM TRIGGERED: TEMPERATURE EXCURSION !!!");
        }
        
        // Track the maximum excursion temperature reached
        if (temp > 8.0 && temp > peakExcursionTemp) peakExcursionTemp = temp;
        if (temp < 2.0 && temp < peakExcursionTemp) peakExcursionTemp = temp;
      }
      
      // 4. Update LCD Display
      lcd.setCursor(0, 0);
      lcd.print("Temp: "); lcd.print(temp, 1); lcd.print("C ");
      
      lcd.setCursor(0, 1);
      if (alarmLatched) {
        lcd.print("ALARM! Ex:");
        lcd.print(peakExcursionTemp, 1);
        lcd.print("C ");
        
        // Sound alarm
        digitalWrite(BUZZER_PIN, HIGH);
      } else {
        lcd.print("Cooler: ");
        lcd.print(digitalRead(RELAY_PIN) == HIGH ? "ON " : "OFF");
        
        digitalWrite(BUZZER_PIN, LOW);
      }
      
      Serial.print("Temp: "); Serial.print(temp, 1);
      Serial.print(" C | Relay: "); Serial.print(digitalRead(RELAY_PIN) == HIGH ? "ON" : "OFF");
      Serial.println(alarmLatched ? " | STATUS: ALARM" : " | STATUS: SAFE");
    } else {
      Serial.println("Sensor read error!");
      lcd.setCursor(0, 0);
      lcd.print("Sensor Offline  ");
    }
    
    lastSampleTime = now;
  }
  
  // 5. Evaluate Alarm Reset Button
  if (resetPressed && alarmLatched) {
    // Only allow clearing the alarm if the temperature has returned to the safe zone
    sensors.requestTemperatures();
    float currentTemp = sensors.getTempCByIndex(0);
    
    if (currentTemp >= TEMP_SAFE_MIN && currentTemp <= TEMP_SAFE_MAX) {
      Serial.println("Alarm cleared. Temperature is back in safe range.");
      alarmLatched = false;
      digitalWrite(BUZZER_PIN, LOW);
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Alarm Cleared   ");
      delay(1500);
      lcd.clear();
    } else {
      Serial.println("Cannot reset alarm. Temperature is still out of safe bounds!");
      // Flash display warning
      lcd.setCursor(0, 1);
      lcd.print("CANNOT RESET!   ");
      delay(1000);
    }
  }
  
  delay(20);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DS18B20**, **Relay**, **Buzzer**, **Button**, and **16x2 I2C LCD** onto the canvas.
2. Wire DS18B20 to **GPIO4**, Relay to **GPIO13**, Buzzer to **GPIO15**, Button to **GPIO12**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the DS18B20 temperature to 10 °C. Watch the buzzer alarm sound and the LCD display `ALARM!`.
5. Slide the temperature back to 5 °C. The buzzer continues to sound (latched).
6. Click the reset button widget. The alarm stops and the LCD returns to normal.

## Expected Output
Serial Monitor:
```
Temp: 5.5 C | Relay: ON | STATUS: SAFE
Temp: 10.2 C | Relay: ON | STATUS: SAFE
!!! ALARM TRIGGERED: TEMPERATURE EXCURSION !!!
Temp: 10.2 C | Relay: ON | STATUS: ALARM
Temp: 5.2 C | Relay: ON | STATUS: ALARM (Still alarm: latched!)
Alarm cleared. Temperature is back in safe range.
```

LCD Display (alarm latched):
```
Temp: 5.2C
ALARM! Ex:10.2C
```

## Expected Canvas Behavior
* At boot, the LCD displays the live temperature.
* Raising the temperature slider above 8.0 °C turns the buzzer widget ON (green) and updates the LCD to `ALARM!`.
* Lowering the temperature to 5.0 °C does not stop the buzzer.
* Pressing the button widget stops the buzzer only if the temperature is in the safe range.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `alarmLatched = true` | Latches the alarm flag once the temperature drifts out of bounds. |
| `peakExcursionTemp` | Records the maximum temperature deviation reached to help staff audit the event. |
| `currentTemp >= TEMP_SAFE_MIN` | Verifies the temperature is back in the safe range before allowing a reset. |

## Hardware & Safety Concept: Alarm Latching in Medical Cold Chains
In medical storage (such as vaccine refrigerators), the temperature must remain constant. If the power cuts out for an hour, the temperature rises and ruins the vaccines. Even if the power returns and the cooler returns the temperature to 4.0 °C, the vaccine has been degraded. If the alarm only sounded while the temperature was high, staff would never know. An **alarm latch** keeps the alarm active until manually reset, ensuring staff are alerted to temperature excursions.

## Try This! (Challenges)
1. **Excursion Data Logger**: Log the start time, end time, and peak temperature of all excursions to an SD card (Project 137).
2. **Compressor Cycle Delay**: Implement a 3-minute delay before restarting the compressor relay to prevent motor damage.
3. **Emergency Battery Backup**: Trigger a low-power sleep mode if the main power rail drops, keeping the alarm active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound during alarm | Buzzer pin misconfigured | Confirm the active buzzer is connected to GPIO 15 |
| Temperature reads -127 °C | OneWire bus pull-up missing | Verify the 4.7 kΩ pull-up resistor is installed between the data line (GPIO 4) and 3.3V |
| Alarm cannot be reset | Temperature still out of bounds | Wait for the temperature slider to return between 2.0 °C and 8.0 °C before pressing the reset button |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [122 - ESP32 Cold Chain Monitor](../intermediate/122-esp32-cold-chain-monitor.md)
- [81 - ESP32 DS18B20 OneWire Temp Sensor Serial Logs](../intermediate/81-esp32-ds18b20-onewire-temp-sensor-serial-logs.md)
- [167 - ESP32 Current Overload Contactor Breaker](167-esp32-current-overload-contactor-breaker.md)
