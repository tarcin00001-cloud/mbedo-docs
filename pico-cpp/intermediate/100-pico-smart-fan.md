# 100 - Pico Smart Fan

Build an automated smart cooling fan that scales motor speed based on temperature and displays status on an LCD.

## Goal
Learn how to read digital temperature sensors (DHT22), scale PWM outputs (DC motor), and display status messages on an I2C LCD simultaneously.

## What You Will Build
An automated smart climate control system:
- **DHT22 Sensor (GP12)**: Monitors room temperature.
- **DC Motor (via L298N)**: Speed scales dynamically (Slow/Medium/Full) matching temperature zones.
- **16x2 I2C LCD (GP4, GP5)**: Displays live temperature and current fan status (e.g. "Temp: 28C | Fan: Mid").
- **L298N Pins**: ENA (GP13), IN1 (GP14), IN2 (GP15).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| DC Motor | `dc_motor` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DHT22 | VCC | 3.3V | Power supply |
| DHT22 | SDA | GP12 | Data signal (requires 10k pull-up to 3.3V) |
| DHT22 | GND | GND | Ground reference |
| L298N | ENA | GP13 | Speed PWM control |
| L298N | IN1 | GP14 | Direction control 1 |
| L298N | IN2 | GP15 | Direction control 2 |
| L298N | GND | GND | Ground return |
| I2C LCD | VCC | 5V | Display power |
| I2C LCD | SDA | GP4 | I2C Data |
| I2C LCD | SCL | GP5 | I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <LiquidCrystal_I2C.h>

const int DHT_PIN = 12;
const int ENA     = 13;
const int IN1     = 14;
const int IN2     = 15;

#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  dht.begin();
  lcd.init();
  lcd.backlight();

  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);

  // Set motor forward
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);

  lcd.setCursor(0, 0);
  lcd.print("Smart Fan Ready");
  delay(1000);
}

void loop() {
  float temp = dht.readTemperature();

  if (!isnan(temp)) {
    int speedPWM = 0;
    lcd.clear();
    
    // Row 0: Temperature
    lcd.setCursor(0, 0);
    lcd.print("Temp: ");
    lcd.print(temp);
    lcd.print(" C");

    // Row 1: Fan Speed Status
    lcd.setCursor(0, 1);
    if (temp < 25.0) {
      speedPWM = 0; // Turn fan OFF
      lcd.print("Fan : OFF");
    } else if (temp >= 25.0 && temp < 30.0) {
      speedPWM = 128; // 50% power
      lcd.print("Fan : MEDIUM");
    } else {
      speedPWM = 255; // 100% power
      lcd.print("Fan : FULL!");
    }

    analogWrite(ENA, speedPWM);
  }

  delay(2000); // DHT22 timing window
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **L298N**, **DC Motor**, and **I2C LCD** onto the canvas.
2. Connect DHT22 to **GP12**, L298N to **GP13-15**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the DHT22 temp slider and observe fan speed and LCD updates.

## Expected Output

Terminal:
```
Simulation active. Smart fan temperature controller online.
```

## Expected Canvas Behavior
| DHT22 Temperature | GP13 (ENA) PWM | Motor Spin Speed | LCD Row 1 Print |
| --- | --- | --- | --- |
| < 25.0 C | 0 | Stopped | `Fan : OFF` |
| 25.0–30.0 C | 128 | Medium speed | `Fan : MEDIUM` |
| > 30.0 C | 255 | Full speed | `Fan : FULL!` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `analogWrite(ENA, speedPWM)` | Sends the speed PWM signal to the L298N enable pin to control the fan's motor speed. |

## Hardware & Safety Concept: Fan Speed Hysteresis
Like thermostats, smart fans use hysteresis to prevent rapid switching when the temperature fluctuates around the threshold (e.g. at 24.9°C to 25.1°C). If the motor toggles ON and OFF continuously every few seconds, it creates electrical noise, wears out relay contacts or motor drivers, and generates annoying clicking sounds. Adding a 1-degree deadband resolves this issue.

## Try This! (Challenges)
1. **Manual Override Switch**: Add a button on GP16 that overrides the automatic control to turn the fan ON at full speed when pressed.
2. **Humidifier Control**: Connect a second relay on GP10 and activate a humidifier if the humidity falls below 45%.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD flashes constantly | Screen clear rate too high | Confirm that `lcd.clear()` and the sensor measurements only execute inside the slow 2-second loop phase. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [54 - Pico Motor Speed](../intermediate/54-pico-motor-speed.md)
- [73 - Pico DHT LCD](../intermediate/73-pico-dht-lcd.md)
