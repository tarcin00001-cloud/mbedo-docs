# 180 - Pico Energy Safety Datalogger

Build an industrial power safety interlock console that measures load currents via an ACS712, opens circuit relays on overcurrent, and logs events over Bluetooth.

## Goal
Learn how to interface analog hall-effect current sensors (ACS712), implement latching overcurrent cutoff algorithms, update LCD status displays, and stream Bluetooth audit reports.

## What You Will Build
An overcurrent safety cutoff console:
- **ACS712 Current Sensor (GP26)**: Monitors the electrical load current.
- **Relay Module (GP10)**: Actuates a simulated circuit breaker.
- **Active Buzzer (GP14)**: Sounds a siren when an overcurrent breach occurs.
- **16x2 I2C LCD (GP4, GP5)**: Displays active current load and breaker status.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits wireless audit warning logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| ACS712 Current Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes (5A ACS712 module) |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| ACS712 Sensor | OUT | GP26 | Analog current load sensor |
| Relay Module | IN | GP10 | Circuit breaker relay |
| Active Buzzer | VCC (+) | GP14 | Overcurrent alarm pin |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int ACS_PIN    = 26;
const int RELAY_PIN  = 10;
const int BUZZ_PIN   = 14;

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Overcurrent trip limit (milliamperes)
const float TRIP_CURRENT_mA = 3500.0;

bool breakerTripped = false;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART

  pinMode(ACS_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZ_PIN, OUTPUT);

  // Keep breaker closed initially (relay HIGH - power ON)
  digitalWrite(RELAY_PIN, HIGH);
  digitalWrite(BUZZ_PIN, LOW);

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Safety Interlock");
  lcd.setCursor(0, 1);
  lcd.print("Breaker Closed  ");
  
  Serial1.println("Power Safety Interlock Online.");
  delay(1500);
}

void loop() {
  int rawACS = analogRead(ACS_PIN);
  
  // Convert current: maps 12-bit ADC (0-4095) to 0-5000mA
  float current_mA = rawACS * 5000.0 / 4095.0;

  // Check overcurrent limit
  if (current_mA > TRIP_CURRENT_mA && !breakerTripped) {
    breakerTripped = true;
    digitalWrite(RELAY_PIN, LOW); // Trip breaker (power OFF)
    
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("!! OVERCURRENT !!");
    lcd.setCursor(0, 1);
    lcd.print("BREAKER TRIPPED ");

    Serial1.print("ALERT: OVERCURRENT TRIP - ");
    Serial1.print(current_mA);
    Serial1.println(" mA");
  }

  // Check for reset command over Bluetooth
  if (Serial1.available()) {
    char cmd = Serial1.read();
    if (cmd == 'R') {
      breakerTripped = false;
      digitalWrite(RELAY_PIN, HIGH); // Re-close breaker (power ON)
      digitalWrite(BUZZ_PIN, LOW);
      
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Breaker Reset   ");
      lcd.setCursor(0, 1);
      lcd.print("Power Restored  ");
      
      Serial1.println("SYSTEM RESET: Breaker Re-closed");
      delay(1500);
    }
  }

  // Sound siren if tripped
  if (breakerTripped) {
    digitalWrite(BUZZ_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZ_PIN, LOW);
    delay(100);
  } else {
    // Normal display monitoring
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Load: ");
    lcd.print(current_mA, 0);
    lcd.print(" mA");

    lcd.setCursor(0, 1);
    lcd.print("Breaker: CLOSED");
    
    delay(500);
  }
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **potentiometer** (for ACS712), **Relay**, **Active Buzzer**, **I2C LCD**, and **HC-05** onto the canvas.
2. Connect ACS712 to **GP26**, Relay to **GP10**, Buzzer to **GP14**, LCD to **GP4/GP5**, and HC-05 to **GP0/GP1**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the current potentiometer above 3500 mA (70% slider), verify the breaker trips, and type `'R'` on Bluetooth to reset it.

## Expected Output

Terminal (Bluetooth):
```
Power Safety Interlock Online.
ALERT: OVERCURRENT TRIP - 3800.00 mA
SYSTEM RESET: Breaker Re-closed
```

## Expected Canvas Behavior
* Normal state (< 3500 mA): LCD reads `Breaker: CLOSED`. Relay remains ON (GP10 HIGH).
* Overcurrent (> 3500 mA): LCD flashes `!! OVERCURRENT !!` / `BREAKER TRIPPED`. Relay turns OFF (GP10 LOW), buzzer sounds. Bluetooth transmits warning log.
* Reset command (`'R'`): Relay turns back ON, buzzer silences.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(RELAY_PIN, LOW)` | Trips the breaker relay, cutting power to the load immediately during overcurrent. |

## Hardware & Safety Concept: Overcurrent Safety Interlocks
Overcurrent safety controllers protect motors and power electronics from shorts and stalls. If currents exceed safety thresholds, the controller trips the circuit breaker relay immediately. These safety controllers must run continuously to protect hardware from damage.

## Try This! (Challenges)
1. **Latching Lockout**: Disable the reset command if an overcurrent event occurs 3 times within 1 minute.
2. **Dynamic Backlight**: Turn OFF the LCD backlight automatically if no alarms are active to save energy.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Breaker trips immediately on boot | Sensor calibration drift | Hall-effect sensors are sensitive to stray magnetic fields. Add a calibration offset in code to zero the reading at startup. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [123 - Pico Energy Safety Monitor](../../expert/123-energy-safety-monitor.md)
- [158 - Pico Solar Datalogger](../advanced/158-pico-solar-datalogger.md)
- [164 - Pico Solar Panel HUD](../advanced/164-pico-solar-panel-hud.md)
