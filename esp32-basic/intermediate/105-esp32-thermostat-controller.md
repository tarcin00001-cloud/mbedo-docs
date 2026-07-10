# 105 - ESP32 Thermostat Controller

Build an automatic thermostat controller that measures temperature using an NTC thermistor (via the Steinhart-Hart equation), controls a heating relay with hysteresis limits, and displays the status on a 16x2 I2C LCD.

## Goal
Learn how to compute calibrated temperature values from raw analog signals in a closed-loop thermostat application, using hysteresis to control heater relays.

## What You Will Build
An NTC thermistor is connected as a voltage divider on GPIO 34. A relay controlling a heater is connected to GPIO 13. The 16x2 I2C LCD displays the live temperature. If the temperature drops below 22.0 °C, the relay turns ON (heater active). Once the temperature rises above 25.0 °C, the relay turns OFF.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| NTC Thermistor (10 kΩ) | `thermistor` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NTC Thermistor | Leg 1 / Leg 2 | 3V3 / GPIO34 | Red / Yellow | Divider top |
| 10 kΩ Resistor | Leg 1 / Leg 2 | GPIO34 / GND | White / Black | Divider bottom (pull-down) |
| Relay Module | IN (Signal) | GPIO13 | Orange | Heater control |
| Relay Module | VCC / GND | 5V / GND | Red / Black | Power rails |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock |

> **Wiring tip:** The NTC thermistor and 10 kΩ fixed resistor form a voltage divider to GPIO 34. Ensure that the fixed resistor value matches the nominal 10 kΩ resistance of the thermistor at 25 °C.

## Code
```cpp
// Thermostat Controller (NTC + Relay + LCD)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <math.h>

const int THERMISTOR_PIN = 34;
const int RELAY_PIN = 13;

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Steinhart-Hart Constants
const float R_FIXED = 10000.0;
const float R_NOMINAL = 10000.0;
const float T_NOMINAL = 25.0;
const float B_COEFF = 3950.0;
const int ADC_MAX = 4095;

// Hysteresis thresholds in Celsius
const float TEMP_LOW = 22.0;  // Turn heater ON below this
const float TEMP_HIGH = 25.0; // Turn heater OFF above this

bool heaterActive = false;

float getCelsiusTemp() {
  int raw = analogRead(THERMISTOR_PIN);
  float voltage = raw * (3.3f / (float)ADC_MAX);
  
  // Calculate resistance from divider formula
  float rTherm = R_FIXED * (3.3f / voltage - 1.0f);
  
  // Steinhart-Hart B-parameter formula
  float steinhart;
  steinhart = rTherm / R_NOMINAL;          // R/Ro
  steinhart = log(steinhart);              // ln(R/Ro)
  steinhart /= B_COEFF;                    // 1/B * ln(R/Ro)
  steinhart += 1.0f / (T_NOMINAL + 273.15f); // + 1/To
  steinhart = 1.0f / steinhart;            // Invert
  return steinhart - 273.15f;              // Convert Kelvin to Celsius
}

void setup() {
  Serial.begin(115200);
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with heater OFF
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Thermostat Station");
  lcd.setCursor(0, 1);
  lcd.print("Initialising...");
  
  delay(1500);
  lcd.clear();
}

void loop() {
  float currentTemp = getCelsiusTemp();
  
  // Apply Hysteresis Control Logic
  if (currentTemp < TEMP_LOW && !heaterActive) {
    digitalWrite(RELAY_PIN, HIGH);
    heaterActive = true;
    Serial.println(">> TEMPERATURE LOW: Heater ON <<");
  } 
  else if (currentTemp > TEMP_HIGH && heaterActive) {
    digitalWrite(RELAY_PIN, LOW);
    heaterActive = false;
    Serial.println(">> TEMPERATURE SATISFIED: Heater OFF <<");
  }
  
  // Update LCD
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(currentTemp, 1);
  lcd.print((char)223);
  lcd.print("C    ");
  
  lcd.setCursor(0, 1);
  lcd.print("Heater: ");
  lcd.print(heaterActive ? "ON " : "OFF");
  
  Serial.print("Temp: "); Serial.print(currentTemp, 1);
  Serial.print(" C | Heater: "); Serial.println(heaterActive ? "ON" : "OFF");
  
  delay(1000); // Poll once per second
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Thermistor**, **Relay**, and **16x2 I2C LCD** onto the canvas.
2. Wire Thermistor output to **GPIO34**, Relay to **GPIO13**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the temperature on the thermistor widget. Set it below 22 °C to turn on the relay, then slide it up past 25 °C to turn it off.

## Expected Output
Serial Monitor:
```
Thermostat Station Initialising...
Temp: 21.4 C | Heater: ON
>> TEMPERATURE LOW: Heater ON <<
Temp: 23.5 C | Heater: ON
Temp: 25.2 C | Heater: OFF
>> TEMPERATURE SATISFIED: Heater OFF <<
```

LCD Display (heating mode active):
```
Temp: 21.4°C
Heater: ON
```

## Expected Canvas Behavior
* At boot, the LCD clears and then displays the current thermistor temperature and heater status.
* When temperature goes below 22.0 °C, the relay widget turns ON.
* The relay widget stays ON until the temperature slider climbs past 25.0 °C.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `getCelsiusTemp()` | Evaluates raw ADC voltage divider outputs using the Steinhart-Hart equation. |
| `currentTemp < TEMP_LOW` | Triggers the heating coil relay when the environment gets cold. |
| `currentTemp > TEMP_HIGH` | Triggers the cutoff limit. |

## Hardware & Safety Concept: Over-Temperature Protection Loops
Automatic climate heaters use dual safety sensors. If a thermistor is disconnected, the analog voltage reads 0V, causing the conversion formula to register an extremely high or low temperature. Thermostats include sanity check filters (e.g. if the temperature reads below -50 °C or above 100 °C, assume sensor failure and shut down the heater immediately).

## Try This! (Challenges)
1. **Sensor Sanity Check Safety**: Modify the code to turn off the relay and flash "Sensor Error" on the LCD if the temp reading is out of realistic bounds (e.g. below 0 °C or above 70 °C).
2. **Dynamic Target Adjuster**: Connect a potentiometer (GPIO 35) to allow the user to set the target temperature dynamically.
3. **Sound Chime Alert**: Add a buzzer on GPIO 15 that plays a short beep when the heater turns on or off.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temperature reads extremely high (> 100 °C) | Thermistor connection issues | Check divider resistors and verify GND and VCC connections |
| Relay switches constantly around 23 °C | Missing hysteresis | Verify separate `TEMP_LOW` and `TEMP_HIGH` variables are used in code |
| LCD display stays blank | Backlight contrast | Adjust contrast potentiometer screw on the back of the LCD backpack |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [35 - ESP32 NTC Thermistor Analog Voltage Logging](../beginner/35-esp32-ntc-thermistor-analog-voltage-logging.md)
- [11 - ESP32 Relay Module Control](../beginner/11-esp32-relay-module-control.md)
- [100 - ESP32 Automatic Smart Fan](100-esp32-automatic-smart-fan.md)
