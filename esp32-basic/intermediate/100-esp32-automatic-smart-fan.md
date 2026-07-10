# 100 - ESP32 Automatic Smart Fan

Build a smart climate control fan that reads temperature from a DHT22 sensor and adjusts a DC motor's cooling speed proportionally, displaying status metrics on a 16x2 I2C LCD.

## Goal
Learn how to create a proportional feedback control loop using temperature inputs to scale DC motor speeds, while displaying status information on an LCD.

## What You Will Build
A DHT22 sensor monitors temperature on GPIO 4. A DC cooling fan is driven through an L298N H-bridge (IN1/IN2: GPIO 18/19, ENA: GPIO 5). If temperature is low (< 26 °C), the fan stays OFF. As it warms up (26 °C to 32 °C), the fan speed increases. The 16x2 I2C LCD displays the temperature and active fan speed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motor (3–12 V) | `dc_motor` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Temp input |
| DHT22 Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| L298N Module | IN1 | GPIO18 | Yellow | Direction control |
| L298N Module | IN2 | GPIO19 | Green | Direction control |
| L298N Module | ENA | GPIO5 | Orange | PWM speed control |
| L298N Module | GND | GND | Black | Ground reference |
| L298N Module | VCC | External 5V/Vin | Red | Motor power |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock |

> **Wiring tip:** Remove the ENA jumper on the L298N module and connect the ENA pin directly to GPIO 5. Power the LCD and motor driver from the 5V Vin rail.

## Code
```cpp
// Automatic Smart Fan Node
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

#define DHTPIN 4
#define DHTTYPE DHT22

const int IN1 = 18;
const int IN2 = 19;
const int ENA_PIN = 5;

const int PWM_CHAN = 0;
const int PWM_FREQ = 2000;
const int PWM_RES = 8;

DHT dht(DHTPIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Temperature thresholds in Celsius
const float TEMP_MIN = 26.0; // Fan starts running at lowest speed
const float TEMP_MAX = 32.0; // Fan runs at 100% speed

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  
  // Set forward rotation direction
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  
  // Initialise PWM
  ledcSetup(PWM_CHAN, PWM_FREQ, PWM_RES);
  ledcAttachPin(ENA_PIN, PWM_CHAN);
  ledcWrite(PWM_CHAN, 0); // Start OFF
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Climate");
  lcd.setCursor(0, 1);
  lcd.print("Control Init...");
  
  dht.begin();
  delay(1500);
  lcd.clear();
}

void loop() {
  // DHT sensors need a 2-second query interval
  delay(2000);
  
  float temp = dht.readTemperature();
  
  if (isnan(temp)) {
    lcd.setCursor(0, 0);
    lcd.print("DHT Sensor Error");
    ledcWrite(PWM_CHAN, 0); // Safety shutdown
    return;
  }
  
  int fanPct = 0;
  int duty = 0;
  
  if (temp < TEMP_MIN) {
    // Too cold — fan OFF
    duty = 0;
    fanPct = 0;
  } 
  else if (temp > TEMP_MAX) {
    // Too hot — fan 100% speed
    duty = 255;
    fanPct = 100;
  } 
  else {
    // Proportional speed ramp between MIN and MAX temp bounds
    // Map 26.0-32.0C to 80-255 PWM duty (starting fan motor needs some minimum torque)
    duty = map((int)(temp * 10), (int)(TEMP_MIN * 10), (int)(TEMP_MAX * 10), 100, 255);
    fanPct = map(duty, 100, 255, 40, 100);
  }
  
  ledcWrite(PWM_CHAN, duty);
  
  // Update LCD
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(temp, 1);
  lcd.print((char)223);
  lcd.print("C    ");
  
  lcd.setCursor(0, 1);
  lcd.print("Fan Speed: ");
  if (fanPct == 0) {
    lcd.print("OFF  ");
  } else {
    lcd.print(fanPct);
    lcd.print("%   ");
  }
  
  Serial.print("Temp: "); Serial.print(temp, 1);
  Serial.print(" C | Fan Duty: "); Serial.print(duty);
  Serial.print(" | Speed: "); Serial.print(fanPct); Serial.println("%");
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, **L298N**, **DC Motor**, and **16x2 I2C LCD** onto the canvas.
2. Wire the DATA pin of the DHT22 to **GPIO4**, motor driver inputs to **GPIO18/GPIO19/GPIO5**, and the LCD SDA/SCL to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the temperature on the DHT22 widget and observe the DC motor spin speed and LCD display change.

## Expected Output
Serial Monitor:
```
Temp: 24.5 C | Fan Duty: 0 | Speed: 0%
Temp: 28.2 C | Fan Duty: 156 | Speed: 67%
Temp: 33.0 C | Fan Duty: 255 | Speed: 100%
```

LCD Display (at moderate heat):
```
Temp: 28.2°C
Fan Speed: 67%
```

## Expected Canvas Behavior
* Below 26 °C, the fan stays stationary and the LCD displays "Fan Speed: OFF".
* Sliding the temperature between 26 °C and 32 °C spins the motor widget at increasing speeds, updating the speed percentage on Row 2.
* Above 32 °C, the motor spins at maximum speed.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `ledcWrite(PWM_CHAN, duty)` | Drives the motor speed via the ENA pin. |
| `map((int)(temp * 10), ...)` | Performs mapping using integer scaling to maintain precision. |
| `100, 255` in map bounds | Sets the minimum starting duty cycle to 100 to ensure the motor has enough torque to spin. |

## Hardware & Safety Concept: Stiction and Minimum Starting Torque
Small DC motors suffer from static friction (stiction). If you supply a low PWM duty cycle (e.g. less than 80 out of 255), the coils cannot generate enough torque to overcome friction, and the motor stalls (humming but not spinning). Stalling motors consume current and generate heat. To prevent this, software loops define a minimum starting threshold value (e.g. 100) below which the motor is shut off.

## Try This! (Challenges)
1. **Humidity Override**: Run the fan at 100% speed if the humidity rises above 85% (condensation protection).
2. **Alert Buzzer**: Sound a warning beep if the temperature climbs past 40 °C.
3. **Manual Calibration Knobs**: Connect two potentiometers to adjust the `TEMP_MIN` and `TEMP_MAX` limits dynamically.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Fan hums but doesn't spin | PWM starting value too low | Increase the minimum duty cycle limit from 100 to 120 in the map function |
| LCD displays corrupt text | I2C wire noise | Keep I2C bus wiring short and route away from motor wires |
| Fan direction is reversed | Output terminals swapped | Swap the OUT1/OUT2 connection wires at the motor driver |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [73 - ESP32 DHT22 Temperature LCD HUD](73-esp32-dht22-temperature-lcd-hud.md)
- [54 - ESP32 DC Motor Speed Scaling](54-esp32-dc-motor-speed-scaling.md)
- [13 - ESP32 5V DC Fan Toggle](../beginner/13-esp32-5v-dc-fan-toggle.md)
