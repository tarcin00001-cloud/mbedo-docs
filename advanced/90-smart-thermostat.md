# 90 - Smart Thermostat

Build a closed-loop thermostat that reads ambient temperature from a DHT22, compares it to a user-adjustable setpoint, and activates or deactivates a relay-controlled fan or heater accordingly.

## Goal
Learn how to implement closed-loop control: reading a sensor, comparing to a target value, and driving an actuator output based on the difference. Use buttons to adjust the target setpoint and display everything on an LCD.

## What You Will Build
Two buttons (Up and Down) allow the user to adjust the temperature setpoint in 1-degree increments. The LCD displays current temperature, humidity, and setpoint. If the current temperature is above the setpoint, the relay activates (turning on a fan or cooling device). If the temperature falls below the setpoint, the relay deactivates.

**Why D2, D3, D4, and D7?** Pins D2/D3 read the Up/Down buttons. Pin D4 communicates with the DHT22. Pin D7 controls the relay.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| 5V Relay | `relay` | Yes | Yes |
| Push Button (x2) | `button` | Yes (x2) | Yes (x2) |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes (DHT pull-up) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DHT22 Sensor | VCC | 5V | Power supply |
| DHT22 Sensor | SDA | D4 | Single-wire data |
| DHT22 Sensor | GND | GND | Ground reference |
| Button (Up) | Pin 1 | D2 | Setpoint increase |
| Button (Up) | Pin 2 | GND | Ground reference |
| Button (Down) | Pin 1 | D3 | Setpoint decrease |
| Button (Down) | Pin 2 | GND | Ground reference |
| Relay | IN | D7 | Control signal |
| Relay | VCC | 5V | Power supply |
| Relay | GND | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

const int DHT_PIN    = 4;
const int BTN_UP     = 2;
const int BTN_DOWN   = 3;
const int RELAY_PIN  = 7;
const int DHT_TYPE   = DHT22;

float setpoint = 25.0; // Default target temperature in Celsius
int lastUpState   = HIGH;
int lastDownState = HIGH;

DHT dht(DHT_PIN, DHT_TYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  dht.begin();
  
  pinMode(BTN_UP,    INPUT_PULLUP);
  pinMode(BTN_DOWN,  INPUT_PULLUP);
  pinMode(RELAY_PIN, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW);
  
  lcd.init();
  lcd.backlight();
  lcd.print("Smart Thermostat");
  delay(1500);
  lcd.clear();
  
  Serial.println("Smart Thermostat Active");
}

void loop() {
  // Read button states
  int upState   = digitalRead(BTN_UP);
  int downState = digitalRead(BTN_DOWN);
  
  // Adjust setpoint on button press (rising edge detection)
  if (upState == LOW && lastUpState == HIGH) {
    setpoint += 1.0;
    delay(150); // Debounce
  }
  if (downState == LOW && lastDownState == HIGH) {
    setpoint -= 1.0;
    delay(150); // Debounce
  }
  
  lastUpState   = upState;
  lastDownState = downState;
  
  // Clamp setpoint to sensible range
  setpoint = constrain(setpoint, 10.0, 40.0);
  
  // Read sensor
  float tempC    = dht.readTemperature();
  float humidity = dht.readHumidity();
  
  if (isnan(tempC) || isnan(humidity)) {
    lcd.setCursor(0, 0);
    lcd.print("Sensor Error!   ");
    return;
  }
  
  Serial.print("Temp: "); Serial.print(tempC);
  Serial.print(" | Set: "); Serial.print(setpoint);
  Serial.print(" | Hum: "); Serial.println(humidity);
  
  // Thermostat logic: activate relay if temperature exceeds setpoint
  if (tempC > setpoint) {
    digitalWrite(RELAY_PIN, HIGH);
    Serial.println("[RELAY] Cooling ON");
  } else {
    digitalWrite(RELAY_PIN, LOW);
    Serial.println("[RELAY] Cooling OFF");
  }
  
  // Update LCD
  lcd.setCursor(0, 0);
  lcd.print("Now:");
  lcd.print(tempC, 0);
  lcd.print("C Set:");
  lcd.print(setpoint, 0);
  lcd.print("C ");
  
  lcd.setCursor(0, 1);
  lcd.print("Hum:");
  lcd.print(humidity, 0);
  lcd.print("% Fan:");
  lcd.print(tempC > setpoint ? "ON " : "OFF");
  
  delay(500);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DHT22 Sensor**, **two Push Buttons**, **Relay**, and **16x2 I2C LCD** onto the canvas.
2. Connect DHT22: **VCC** to **5V**, **SDA** to **D4**, **GND** to **GND**.
3. Connect Up Button: **pin 1** to **D2**, **pin 2** to **GND**.
4. Connect Down Button: **pin 1** to **D3**, **pin 2** to **GND**.
5. Connect Relay: **IN** to **D7**, **VCC** to **5V**, **GND** to **GND**.
6. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
7. Paste the code into the editor.
8. Use the default Arduino interpreted runtime.
9. Click **Run**.
10. Press Up/Down buttons to change setpoint. Adjust DHT22 slider above setpoint and watch the relay activate.

## Expected Output

Terminal:
```
Smart Thermostat Active
Temp: 28.0 | Set: 25.0 | Hum: 65.0
[RELAY] Cooling ON
```

LCD Display:
```
Now:28C Set:25C
Hum:65% Fan:ON
```

## Expected Canvas Behavior

| DHT22 Temp | Setpoint | Comparison | Relay State | Fan Display |
| --- | --- | --- | --- | --- |
| 20 C | 25 C | Below | Open (LOW) | `Fan:OFF` |
| 28 C | 25 C | Above | Closed (HIGH) | `Fan:ON` |
| 25 C | 25 C | Equal | Open (LOW) | `Fan:OFF` |

## Code Walkthrough

| Expression | What It Does |
| --- | --- |
| `setpoint = constrain(setpoint, 10.0, 40.0)` | Prevents the user from pressing buttons to set physically unrealistic temperatures (below 10 C or above 40 C). |
| `tempC > setpoint ? "ON " : "OFF"` | Ternary operator used inline to select the correct display string. |

## Hardware & Safety Concept: On/Off (Bang-Bang) Control
This thermostat implements the simplest form of closed-loop control, called **bang-bang control** (named after the relay clicking sound). The output switches fully ON or fully OFF when the sensor crosses the setpoint.
- Real thermostats add a **deadband** (hysteresis) to prevent the relay from rapidly switching ON and OFF when the temperature hovers exactly at the setpoint (called **chatter** or **hunting**). For example: turn ON at 26 C, but only turn OFF once it falls below 24 C.

## Try This! (Challenges)
1. **Hysteresis**: Implement a 2 C deadband. Activate the relay when `tempC > setpoint + 1` and deactivate when `tempC < setpoint - 1`.
2. **Setpoint Memory**: Store the setpoint in EEPROM so it is restored after a power cycle (use `EEPROM.put()` and `EEPROM.get()`).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Setpoint changes too fast when button held | Missing debounce or edge detection | Verify that `lastUpState`/`lastDownState` are compared on each cycle and that debounce delay is present. |
| Relay chatters rapidly at setpoint | No hysteresis | Add the 2 C deadband described in the challenges section. |

## Mode Notes
These patterns (DHT22 reads, button edge detection, relay control, and LCD output) are supported by MbedO interpreted mode.

## Related Projects
- [50 - High Temp Fan](../intermediate/50-high-temp-fan.md)
- [37 - Night Fan Relay](../intermediate/37-night-fan-relay.md)
- [91 - Indoor Air Station](91-indoor-air-station.md)
