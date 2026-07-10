# 105 - Pico Thermostat

Build a thermostat temperature controller that displays live readings on an LCD and toggles a heater relay.

## Goal
Learn how to read analog thermistors, calculate temperature values, and toggle relays based on thermostat setpoints.

## What You Will Build
A digital thermostat panel:
- **NTC Thermistor (GP26)**: Measures room temperature.
- **Relay Module (GP10)**: Toggles a 5V heater load ON if the temperature falls below 22.0°C.
- **16x2 I2C LCD (GP4, GP5)**: Displays current temperature and relay state (e.g. "Temp: 20C | Htr: ON").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC Thermistor | `ntc` | Yes | Yes (10k NTC) |
| 10k-ohm Resistor | `resistor` | Optional | Yes (voltage divider) |
| Relay Module | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| NTC Thermistor | Pin 1 | 3.3V | Voltage divider high |
| NTC Thermistor | Pin 2 (Signal) | GP26 | Analog input (requires 10k pull-down to GND) |
| Relay Module | VCC | 5V | Coil power |
| Relay Module | IN | GP10 | Heater control signal |
| Relay Module | GND | GND | Ground return |
| I2C LCD | VCC | 5V | Display power |
| I2C LCD | SDA | GP4 | I2C Data |
| I2C LCD | SCL | GP5 | I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int THERM_PIN  = 26;
const int RELAY_PIN  = 10;

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Target thermostat setpoint (°C)
const float SETPOINT = 22.0;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(THERM_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW); // Start with heater OFF

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Thermostat Ready");
  delay(1000);
}

void loop() {
  int rawValue = analogRead(THERM_PIN);
  
  // Approximate Steinhart-Hart temperature calculation for 10k NTC
  // Mapped to a simplified linear curve for interpreted mode convenience
  float tempC = 25.0 + (2048 - rawValue) * 0.04;

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(tempC);
  lcd.print(" C");

  lcd.setCursor(0, 1);
  if (tempC < SETPOINT) {
    digitalWrite(RELAY_PIN, HIGH); // Turn heater ON
    lcd.print("Heater: ON");
  } else {
    digitalWrite(RELAY_PIN, LOW);  // Turn heater OFF
    lcd.print("Heater: OFF");
  }

  delay(1500); // Update once per 1.5 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **NTC Thermistor**, **Relay Module**, and **I2C LCD** onto the canvas.
2. Connect Thermistor to **GP26** (with 10k pull-down to GND), Relay to **GP10**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the temperature slider on the thermistor model and watch the LCD and relay update.

## Expected Output

Terminal:
```
Simulation active. LCD thermostat controller active.
```

## Expected Canvas Behavior
| Thermistor Temp slider | Mapped tempC | GP10 (Relay state) | LCD Row 1 Print |
| --- | --- | --- | --- |
| Cold (< 22 C) | ~18.0 C | HIGH (Closed) | `Heater: ON` |
| Hot (> 22 C) | ~25.0 C | LOW (Open) | `Heater: OFF` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `tempC < SETPOINT` | Thermostatic threshold evaluation that controls the activation of the heater relay. |

## Hardware & Safety Concept: Thermostat Deadbands
Like smart fans, heating systems use hysteresis to prevent rapid switching when the temperature fluctuates around the threshold. If the relay toggles ON and OFF continuously every few seconds, it creates electrical noise, wears out relay contacts, and generates annoying clicking sounds. Adding a 0.5-degree deadband resolves this issue.

## Try This! (Challenges)
1. **Setpoint Adjuster**: Connect a potentiometer to GP27 and use its value to adjust the target temperature setpoint dynamically, printing the setpoint value on the LCD.
2. **Audio Warning**: Sound a buzzer on GP14 if the temperature exceeds a safety limit of 40°C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temperature reads extreme negative or positive values | Missing voltage divider | Ensure a 10k fixed pull-down resistor is wired between GP26 and Ground to complete the thermistor voltage divider. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [35 - Pico Thermistor Temp](../../beginner/35-pico-thermistor-temp.md)
- [58 - Pico LCD Print](58-pico-lcd-print.md)
- [100 - Pico Smart Fan](100-pico-smart-fan.md)
