# 140 - Pico Smart Thermostat

Build an advanced smart thermostat that monitors room temperature, accepts keypad inputs to adjust target setpoints, and toggles a heater relay.

## Goal
Learn how to read analog thermistors, parse keypad input keys without arrays/loops, and display target setpoints on an LCD.

## What You Will Build
An adjustable digital thermostat panel:
- **NTC Thermistor (GP26)**: Measures room temperature.
- **Relay Module (GP10)**: Toggles a heater ON if the temperature falls below the target setpoint.
- **4x4 Keypad (GP2, GP3, GP6, GP7, GP8, GP9, GP20, GP21)**: Pressing `'A'` increments the target setpoint by 1°C, and `'B'` decrements it by 1°C.
- **16x2 I2C LCD (GP4, GP5)**: Displays the current temperature, target setpoint, and heater status.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC Thermistor | `ntc` | Yes | Yes (10k NTC) |
| 10k-ohm Resistor | `resistor` | Optional | Yes (voltage divider) |
| Relay Module | `relay` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| NTC Thermistor | Signal (Pin 2) | GP26 | Analog temperature input (with 10k pull-down) |
| Relay Module | IN | GP10 | Heater control switch |
| Keypad Rows | R1 - R4 | GP2 / GP3 / GP6 / GP7 | Output scanning rows |
| Keypad Cols | C1 - C4 | GP8 / GP9 / GP20 / GP21 | Input columns (pull-up) |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int THERM_PIN  = 26;
const int RELAY_PIN  = 10;

// Keypad pins
const int R1 = 2; const int R2 = 3; const int R3 = 6; const int R4 = 7;
const int C1 = 8; const int C2 = 9; const int C3 = 20; const int C4 = 21;

LiquidCrystal_I2C lcd(0x27, 16, 2);

int targetSetpoint = 22; // Default setpoint 22°C

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(THERM_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start heater OFF

  // Setup keypad pins
  pinMode(R1, OUTPUT); pinMode(R2, OUTPUT); pinMode(R3, OUTPUT); pinMode(R4, OUTPUT);
  pinMode(C1, INPUT_PULLUP); pinMode(C2, INPUT_PULLUP); pinMode(C3, INPUT_PULLUP); pinMode(C4, INPUT_PULLUP);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Thermostat");
  delay(1000);
}

void loop() {
  int rawValue = analogRead(THERM_PIN);
  
  // Approximate linear temperature calculation for 10k NTC
  float tempC = 25.0 + (2048 - rawValue) * 0.04;

  // Read keypad input key
  char key = scanKeypad();
  if (key != '\0') {
    if (key == 'A') {
      targetSetpoint = targetSetpoint + 1; // Increment setpoint
      delay(250); // Debounce delay
    } 
    else if (key == 'B') {
      targetSetpoint = targetSetpoint - 1; // Decrement setpoint
      delay(250); // Debounce delay
    }
  }

  // Update display
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(tempC, 1);
  lcd.print("C");

  lcd.setCursor(0, 1);
  lcd.print("Set: ");
  lcd.print(targetSetpoint);
  lcd.print("C | ");

  // Thermostat relay actuation
  if (tempC < targetSetpoint) {
    digitalWrite(RELAY_PIN, HIGH); // Turn heater ON
    lcd.print("Htr: ON ");
  } else {
    digitalWrite(RELAY_PIN, LOW);  // Turn heater OFF
    lcd.print("Htr: OFF");
  }

  delay(200);
}

char scanKeypad() {
  char pressedKey = '\0';

  // Row 1
  digitalWrite(R1, LOW); digitalWrite(R2, HIGH); digitalWrite(R3, HIGH); digitalWrite(R4, HIGH);
  if (digitalRead(C1) == LOW) { pressedKey = '1'; }
  if (digitalRead(C2) == LOW) { pressedKey = '2'; }
  if (digitalRead(C3) == LOW) { pressedKey = '3'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'A'; }

  // Row 2
  digitalWrite(R1, HIGH); digitalWrite(R2, LOW); digitalWrite(R3, HIGH); digitalWrite(R4, HIGH);
  if (digitalRead(C1) == LOW) { pressedKey = '4'; }
  if (digitalRead(C2) == LOW) { pressedKey = '5'; }
  if (digitalRead(C3) == LOW) { pressedKey = '6'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'B'; }

  // Row 3
  digitalWrite(R1, HIGH); digitalWrite(R2, HIGH); digitalWrite(R3, LOW); digitalWrite(R4, HIGH);
  if (digitalRead(C1) == LOW) { pressedKey = '7'; }
  if (digitalRead(C2) == LOW) { pressedKey = '8'; }
  if (digitalRead(C3) == LOW) { pressedKey = '9'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'C'; }

  // Row 4
  digitalWrite(R1, HIGH); digitalWrite(R2, HIGH); digitalWrite(R3, HIGH); digitalWrite(R4, LOW);
  if (digitalRead(C1) == LOW) { pressedKey = '*'; }
  if (digitalRead(C2) == LOW) { pressedKey = '0'; }
  if (digitalRead(C3) == LOW) { pressedKey = '#'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'D'; }

  return pressedKey;
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **NTC Thermistor**, **4x4 Keypad**, **Relay**, and **I2C LCD** onto the canvas.
2. Connect Thermistor to **GP26**, Relay to **GP10**, Keypad to Row/Col pins, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Press `'A'` on the keypad to increase the target temperature, or `'B'` to decrease it, and watch the relay toggle status.

## Expected Output

Terminal:
```
Simulation active. Adjustable smart thermostat console online.
```

## Expected Canvas Behavior
* Startup: LCD reads `Temp: 24.0C` / `Set: 22C | Htr: OFF`. Relay is OFF.
* Press `'A'` three times: Target Setpoint increases to 25°C. LCD updates to `Set: 25C | Htr: ON`. Relay clicks ON immediately.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `targetSetpoint = targetSetpoint + 1` | Increments the target temperature setpoint variable on keypad input to adjust comfort zones. |

## Hardware & Safety Concept: Industrial Setpoint Management
Industrial climate controllers use keypads or buttons to adjust temperature setpoints. To prevent unauthorized users from tampering with setpoints, real HVAC control interfaces require entering an authorization password (PIN) before allowing changes to the temperature configuration.

## Try This! (Challenges)
1. **Critical Overheat Alarm**: Sound a buzzer on GP14 if the temperature exceeds a safety limit of 40°C.
2. **Dynamic Backlight**: Turn OFF the LCD backlight automatically if no keys are pressed for 15 seconds to save energy.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temperature reads static values | Voltage divider error | Ensure a 10k fixed pull-down resistor is wired between GP26 and Ground to complete the thermistor voltage divider. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [105 - Pico Thermostat](../intermediate/105-pico-thermostat.md)
- [124 - Pico Smart Lock](124-pico-smart-lock.md)
