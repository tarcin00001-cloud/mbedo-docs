# 200 - Pico Smart Thermostat Datalogger OLED

Build an adjustable smart HVAC thermostat that saves temperature setpoints in flash (EEPROM), controls a heater relay, displays diagnostics on an OLED, and streams climate logs over Bluetooth.

## Goal
Learn how to read NTC thermistors, scan keypads, read/write EEPROM, actuate relays, design graphical OLED dashboards, and stream telemetry CSV data packets wirelessly.

## What You Will Build
An adjustable digital thermostat panel with wireless audit logs and OLED HUD:
- **NTC Thermistor (GP26)**: Measures room temperature.
- **Relay Module (GP10)**: Toggles a heater ON when the temperature falls below the target.
- **4x4 Keypad**: Pressing `'A'` increments the target setpoint, and `'B'` decrements it.
- **EEPROM (Flash)**: Saves the adjusted setpoint temperature at address `40`, ensuring target parameters persist across power cycles.
- **SSD1306 OLED (GP4, GP5)**: Displays the current temperature, target setpoint, and active heater status.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits wireless weather logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| NTC Thermistor | `ntc` | Yes | Yes (10k NTC) |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| 10k-ohm Resistors | `resistor` | Optional | Yes (voltage divider) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| NTC Thermistor | Signal (Pin 2) | GP26 | Analog temperature input (with 10k pull-down) |
| Relay Module | IN | GP10 | Heater control switch |
| Keypad Rows | R1 - R4 | GP2 / GP3 / GP6 / GP7 | Output scanning rows |
| Keypad Cols | C1 - C4 | GP8 / GP9 / GP20 / GP21 | Input columns (pull-up) |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <EEPROM.h>

const int THERM_PIN  = 26;
const int RELAY_PIN  = 10;

// Keypad pins
const int R1 = 2; const int R2 = 3; const int R3 = 6; const int R4 = 7;
const int C1 = 8; const int C2 = 9; const int C3 = 20; const int C4 = 21;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

int targetSetpoint = 22; // Default setpoint 22°C

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(THERM_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start heater OFF

  // Setup keypad pins
  pinMode(R1, OUTPUT); pinMode(R2, OUTPUT); pinMode(R3, OUTPUT); pinMode(R4, OUTPUT);
  pinMode(C1, INPUT_PULLUP); pinMode(C2, INPUT_PULLUP); pinMode(C3, INPUT_PULLUP); pinMode(C4, INPUT_PULLUP);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  EEPROM.begin(512);

  // Read targetSetpoint from EEPROM address 40
  int val = EEPROM.read(40);
  if (val >= 10 && val <= 40) { // Limit valid temperature bounds (10°C to 40°C)
    targetSetpoint = val;
  }

  // Print CSV Header over Bluetooth
  Serial1.println("Temp_C,Setpoint_C,Heater_ON");
  delay(1000);
}

void loop() {
  int rawValue = analogRead(THERM_PIN);
  
  // Approximate linear temperature calculation for 10k NTC
  float tempC = 25.0 + (2048 - rawValue) * 0.04;

  bool setpointChanged = false;

  // Read keypad input key
  char key = scanKeypad();
  if (key != '\0') {
    if (key == 'A') {
      targetSetpoint = targetSetpoint + 1; // Increment setpoint
      setpointChanged = true;
      delay(250); // Debounce delay
    } 
    else if (key == 'B') {
      targetSetpoint = targetSetpoint - 1; // Decrement setpoint
      setpointChanged = true;
      delay(250); // Debounce delay
    }
  }

  // Save new setpoint to EEPROM if changed
  if (setpointChanged) {
    EEPROM.write(40, targetSetpoint);
    EEPROM.commit();
  }

  bool heaterActive = false;

  // Thermostat relay actuation
  if (tempC < targetSetpoint) {
    digitalWrite(RELAY_PIN, HIGH); // Turn heater ON
    heaterActive = true;
  } else {
    digitalWrite(RELAY_PIN, LOW);  // Turn heater OFF
    heaterActive = false;
  }

  // Update OLED display
  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(18, 3);
  display.print("THERMOSTAT HUD");

  // Display stats
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  
  display.setCursor(8, 20);
  display.print("Current: ");
  display.print(tempC, 1);
  display.print(" C");

  display.setCursor(8, 34);
  display.print("Set    : ");
  display.print(targetSetpoint);
  display.print(" C");

  display.setCursor(8, 48);
  display.print("Heater : ");
  display.print(heaterActive ? "ACTIVE/ON" : "STANDBY/OFF");

  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  // Stream CSV logs over Bluetooth
  Serial1.print(tempC, 2);
  Serial1.print(",");
  Serial1.print(targetSetpoint);
  Serial1.print(",");
  Serial1.println(heaterActive ? 1 : 0);

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
1. Drag the **Raspberry Pi Pico**, **NTC Thermistor**, **4x4 Keypad**, **Relay**, **SSD1306 OLED**, and **HC-05** onto the canvas.
2. Connect Thermistor to **GP26**, Relay to **GP10**, Keypad to Row/Col pins, OLED to **GP4/GP5**, and HC-05 to **GP0/GP1**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Press `'A'` on the keypad to increase the target temperature, and verify that the Bluetooth log and OLED display update.

## Expected Output

Terminal (Bluetooth):
```
Temp_C,Setpoint_C,Heater_ON
24.00,22,0
24.00,25,1
```

## Expected Canvas Behavior
* Startup: OLED reads `Current: 24.0 C` / `Set: 22 C` / `Heater: STANDBY/OFF`. Bluetooth streams starting telemetry log.
* Press `'A'` three times: Target Setpoint increases to 25°C. OLED updates to `Set: 25 C` and `Heater: ACTIVE/ON`. Relay clicks ON immediately. Bluetooth prints `24.00,25,1`.
* Restart Simulation: Thermostat boots up with the target temperature loaded from memory.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.println(heaterActive ? 1 : 0)` | Sends the active heating switch state (1 = ON, 0 = OFF) wirelessly to the Bluetooth host. |

## Hardware & Safety Concept: Industrial Setpoint Management
Industrial climate controllers use keypads or buttons to adjust temperature setpoints. To prevent unauthorized users from tampering with setpoints, real HVAC control interfaces require entering an authorization password (PIN) before allowing changes to the temperature configuration.

## Try This! (Challenges)
1. **Critical Overheat Alarm**: Sound a buzzer on GP14 if the temperature exceeds a safety limit of 40°C.
2. **Display Sleep**: Turn OFF the OLED screen automatically if no keys are pressed for 15 seconds to save battery.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Setpoint resets to 22°C on reboot | `EEPROM.commit()` missing | Ensure `EEPROM.commit()` is called after `EEPROM.write()` to save the value to physical flash memory. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [105 - Pico Thermostat](../intermediate/105-pico-thermostat.md)
- [140 - Pico Smart Thermostat](../advanced/140-pico-smart-thermostat.md)
- [169 - Pico Smart Thermostat EEPROM](../advanced/169-pico-smart-thermostat-eeprom.md)
- [185 - Pico Smart Thermostat Datalogger](185-pico-smart-thermostat-datalogger.md)
