# 170 - Pico Gas Warning Datalogger

Build an industrial gas and fire safety console that monitors gas levels and infrared flame indicators, shuts off safety valves, and streams CSV logs to Serial.

## Goal
Learn how to monitor multiple analog inputs, update warning summaries on I2C LCDs, and stream structured CSV logs to the Serial Monitor.

## What You Will Build
An industrial gas warning monitor:
- **MQ-2 Gas Sensor (GP26)**: Measures gas concentrations.
- **LDR Flame Sensor (GP27)**: Detects fire infrared light.
- **Relay Module (GP10)**: Closes an emergency solenoid valve during alerts.
- **Active Buzzer (GP14)**: Sounds a warning siren.
- **16x2 I2C LCD (GP4, GP5)**: Displays live gas indices and safety status.
- **Serial Datalogger**: Streams CSV log lines (e.g. `Gas,Fire,Valve_state`) every 3 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MQ-2 Gas Sensor | `gas_sensor` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes (configured as flame IR receiver) |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MQ-2 Sensor | AO | GP26 | Gas concentration input |
| LDR Sensor | AO (Signal) | GP27 | Flame index input |
| Relay Module | IN | GP10 | Gas supply solenoid valve |
| Active Buzzer | VCC (+) | GP14 | Alarm siren pin |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int MQ2_PIN    = 26;
const int FLAME_PIN  = 27;
const int RELAY_PIN  = 10;
const int BUZZER_PIN = 14;

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Limits
const int GAS_LIMIT   = 1800; // Gas leak raw threshold
const int FLAME_LIMIT = 1500; // Flame IR threshold (drops low near fire)

void setup() {
  Serial.begin(9600);

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(MQ2_PIN, INPUT);
  pinMode(FLAME_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  // Keep gas valve open at startup (relay LOW)
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Gas Logger");
  lcd.setCursor(0, 1);
  lcd.print("System Active  ");

  // Print CSV Header to Serial
  Serial.println("Gas_idx,Flame_idx,Valve_shutdown");
  delay(1500);
}

void loop() {
  int gasVal = analogRead(MQ2_PIN);
  int flameVal = analogRead(FLAME_PIN);

  bool gasLeak = (gasVal > GAS_LIMIT);
  bool fireDetect = (flameVal < FLAME_LIMIT);

  lcd.clear();

  if (gasLeak || fireDetect) {
    // EMERGENCY SHUTDOWN ACTIVE
    digitalWrite(RELAY_PIN, HIGH); // Shut off gas supply solenoid valve
    
    lcd.setCursor(0, 0);
    if (gasLeak && fireDetect) {
      lcd.print("!! FIRE & GAS !!");
    } else if (gasLeak) {
      lcd.print("!! GAS LEAK !!");
    } else {
      lcd.print("!! FIRE ALERT !!");
    }

    lcd.setCursor(0, 1);
    lcd.print("VALVE: SHUTDOWN ");

    // Pulse siren
    digitalWrite(BUZZER_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZER_PIN, LOW);
    delay(100);
  } else {
    // Normal Secure operation
    digitalWrite(RELAY_PIN, LOW); // Keep gas supply open
    digitalWrite(BUZZER_PIN, LOW);

    lcd.setCursor(0, 0);
    lcd.print("Gas: ");
    lcd.print(gasVal);
    lcd.print(" index");

    lcd.setCursor(0, 1);
    lcd.print("Valve: OPEN     ");
  }

  // Stream CSV logs to Serial
  Serial.print(gasVal);
  Serial.print(",");
  Serial.print(flameVal);
  Serial.print(",");
  Serial.println((gasLeak || fireDetect) ? 1 : 0);

  delay(3000); // Log once every 3 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MQ-2 Sensor**, **LDR**, **Relay**, **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect MQ-2 to **GP26**, LDR to **GP27**, Relay to **GP10**, Buzzer to **GP14**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the Gas PPM slider or light intensity to trigger the emergency shutoff valve, and watch the CSV logs stream.

## Expected Output

Terminal:
```
Gas_idx,Flame_idx,Valve_shutdown
800,4095,0
1900,4095,1
800,1200,1
```

## Expected Canvas Behavior
* Normal state: LCD reads `Gas: 800 index` / `Valve: OPEN`. Relay is OFF. CSV prints `800,4095,0`.
* Gas leak (> 1800): LCD reads `!! GAS LEAK !!` / `VALVE: SHUTDOWN`. Relay turns ON, buzzer beeps. CSV prints `1900,4095,1`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `gasLeak \|\| fireDetect` | Logical OR check that triggers the emergency shutoff valve if either hazard is detected. |

## Hardware & Safety Concept: Fail-Safe Valve Controls
In gas line safety systems, solenoid valves are designed to be **Normally Closed (NC)**. This means they require continuous electrical power to stay OPEN. If power is cut off during a fire or gas leak, the valve automatically snaps closed by spring pressure, preventing gas from leaking. Storing warning logs locally or streaming them over serial provides diagnostic data to identify when and why shutdowns occurred.

## Try This! (Challenges)
1. **Latching Lockout**: If an alarm is triggered, lock the valve in the SHUTDOWN state until a physical reset button (GP16) is pressed.
2. **Alert Indicator**: Connect a warning LED on GP15 and flash it in sync with the buzzer alarm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay oscillates ON/OFF rapidly | Sensor noise | Add a small hysteretic delay or check averaging loops to prevent minor sensor noise from toggling the valve. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [98 - Pico Gas Leak](../intermediate/98-pico-gas-leak.md)
- [132 - Pico Gas Warning](132-pico-gas-warning.md)
- [147 - Pico Gas OLED Alarm](147-pico-gas-oled-alarm.md)
