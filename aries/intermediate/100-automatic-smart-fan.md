# 100 - Automatic Smart Fan

Design an automated climate control fan that reads temperature and humidity from a DHT22 sensor, triggering a DC motor based on temperature thresholds and displaying system statuses on an I2C LCD display.

## Goal
Learn how to read digital environmental data, operate a DC motor based on temperature conditions, and update a character LCD using a state-tracking algorithm that avoids C++ loops.

## What You Will Build
A smart home temperature-regulated fan console. The system continuously polls the room's temperature and humidity. If the temperature exceeds 28°C, the DC fan motor turns on automatically. The current climate and fan status (RUNNING or STOPPED) are shown on the LCD screen.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Climate Sensor | `dht22` | Yes | Yes |
| 5V DC Motor / Fan | `motor` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | VCC | 3V3 | Red | Sensor operating voltage (3.3V) |
| DHT22 Sensor | GND | GND | Black | Ground reference |
| DHT22 Sensor | SDA (Data) | GPIO 12 | Green | Single-wire data line |
| DC Motor | Positive (+) | GPIO 14 | Orange | Control signal pin |
| DC Motor | Negative (-) | GND | Black | Ground reference |
| I2C LCD | VCC | 5V | Red | LCD power supply (5V) |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** A physical DC motor draws too much current to be powered directly from a GPIO pin. In real hardware, use a transistor (like a PN2222) or an H-bridge motor driver to switch external power to the motor, controlled by GPIO 14.

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

const int DHT_PIN = 12;
#define DHTTYPE DHT22

const int FAN_PIN = 14; // DC Motor on GPIO 14
const float TEMP_THRESHOLD = 28.0; // Target temperature in Celsius

DHT dht(DHT_PIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

float lastTemp = -1.0;
float lastHum = -1.0;
int lastFanState = -1;

void setup() {
  Wire.begin();
  dht.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Fan Sys");

  pinMode(FAN_PIN, OUTPUT);
  digitalWrite(FAN_PIN, LOW); // Start with fan off
}

void loop() {
  float temp = dht.readTemperature();
  float hum = dht.readHumidity();

  // Validate that the sensor returned values
  if (!isnan(temp) && !isnan(hum)) {
    int fanState = 0;
    
    if (temp >= TEMP_THRESHOLD) {
      fanState = 1;
      digitalWrite(FAN_PIN, HIGH); // Activate fan
    } else {
      fanState = 0;
      digitalWrite(FAN_PIN, LOW);  // Turn off fan
    }

    // Refresh screen only on changes to eliminate screen flicker
    if (temp != lastTemp || hum != lastHum || fanState != lastFanState) {
      lcd.setCursor(0, 0);
      lcd.print("T:");
      lcd.print(temp, 1);
      lcd.print("C H:");
      lcd.print(hum, 1);
      lcd.print("%");

      lcd.setCursor(0, 1);
      if (fanState == 1) {
        lcd.print("FAN: RUNNING    ");
      } else {
        lcd.print("FAN: STOPPED    ");
      }

      lastTemp = temp;
      lastHum = hum;
      lastFanState = fanState;
    }
  }

  delay(2000); // Wait 2 seconds between DHT22 readings
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DHT22 Climate Sensor**, **DC Motor**, and **I2C LCD Display** components onto the canvas.
2. Wire the DHT22: **VCC** to **3V3**, **GND** to **GND**, and **SDA** to **GPIO 12**.
3. Wire the DC Motor: **Positive** to **GPIO 14** and **Negative** to **GND**.
4. Wire the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Temp: 29.5 C | Humidity: 60.1% | Fan: ON
```

## Expected Canvas Behavior
* Increasing the temperature slider on the simulated DHT22 sensor above 28.0°C turns the DC motor.
* The LCD displays live temperature and humidity metrics alongside the `FAN: RUNNING` status indicator.
* Moving the temperature slider below 28.0°C causes the DC motor to stop spinning and changes the LCD status to `FAN: STOPPED`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `dht.begin()` | Starts communication with the DHT22 sensor. |
| `dht.readTemperature()` | Fetches temperature from the sensor via the single-wire protocol. |
| `temp >= TEMP_THRESHOLD` | Evaluates if the current temperature exceeds the trigger threshold (28.0°C). |
| `digitalWrite(FAN_PIN, HIGH)` | Drives GPIO 14 high to activate the fan. |
| `lcd.print(temp, 1)` | Prints temperature formatting with a single decimal place. |

## Hardware & Safety Concept
* **Back-EMF Snubber Diode**: DC motors are inductive loads. When turned OFF, the magnetic field in the motor windings collapses, inducing a high-voltage reverse spike (back-EMF) that can destroy the switching transistor and damage the microcontroller. Always wire a flyback diode (like a 1N4007) in parallel with the motor terminal, pointing away from ground, to safely dissipate this spike.
* **DHT22 Timing Requirements**: The DHT22 requires a minimum of 2 seconds between reads to recharge its internal sensor elements. Querying the sensor faster will result in old cached values or communication timeouts.

## Try This! (Challenges)
1. **Dynamic Fan Speeds**: Switch the fan control to a PWM pin (GPIO 14 supports PWM). Drive the fan at 50% speed (`analogWrite(14, 128)`) between 28°C and 32°C, and 100% speed (`analogWrite(14, 255)`) above 32°C.
2. **Humidity Over-ride**: If the humidity levels climb above 80% (indicating very stuffy air), force the fan to run regardless of the current temperature.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD does not update or remains blank | Wrong I2C address | Ensure the address parameter in `LiquidCrystal_I2C lcd(0x27, 16, 2)` matches your LCD device address. |
| Fan stays off even when DHT22 is hot | Pin configuration mismatch | Confirm the motor positive terminal connects to ARIES GPIO 14. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [58 - 16x2 I2C LCD Hello World](58-16x2-i2c-lcd-hello-world.md)
- [67 - Potentiometer Speed DC Motor (with LCD)](67-potentiometer-speed-dc-motor.md)
