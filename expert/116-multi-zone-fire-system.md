# 116 - Multi-Zone Fire System

Build a comprehensive safety controller that monitors two separate hazard inputs: a digital flame sensor (Zone 1) and an analog gas sensor (Zone 2). If either hazard is detected, a buzzer triggers, a relay cuts power, and the LCD displays which zone triggered the alarm.

## Goal
Learn how to monitor multiple safety zones in parallel using digital and analog sensor inputs, perform threshold checks, and coordinate multiple outputs (buzzer, relay, LCD) without blocking program execution.

## What You Will Build
An active hazard monitoring console:
- **Zone 1 (Flame)**: Monitored via a digital input. If `LOW` (flame detected), triggers safety cut-off.
- **Zone 2 (Gas)**: Monitored via an analog input. If value > 400, triggers safety cut-off.
- **LCD Display**: Shows status of both zones (e.g. `Z1:OK Z2:OK` or `ALERT Z1:FLAME`).
- **Relay & Buzzer**: Actuates immediately to cut power and sound alarm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Flame Sensor | `flame_sensor` | Yes | Yes |
| MQ-2 Gas Sensor | `mq2` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Flame Sensor | VCC | 5V | Power supply |
| Flame Sensor | DO | D2 | Digital flame alert (active-LOW) |
| Flame Sensor | GND | GND | Ground reference |
| MQ-2 Sensor | VCC | 5V | Power supply |
| MQ-2 Sensor | AO | A1 | Analog gas level |
| MQ-2 Sensor | GND | GND | Ground reference |
| Relay Module | VCC | 5V | Power supply |
| Relay Module | IN | D8 | Relay control (HIGH = Alarm / Cutoff) |
| Relay Module | GND | GND | Ground reference |
| Active Buzzer | VCC | D7 | Buzzer drive pin |
| Active Buzzer | GND | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int FLAME_PIN  = 2;
const int GAS_PIN    = A1;
const int BUZZER_PIN = 7;
const int RELAY_PIN  = 8;

const int GAS_THRESHOLD = 400;

LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  pinMode(FLAME_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);

  digitalWrite(RELAY_PIN, LOW); // System normal, relay closed
  digitalWrite(BUZZER_PIN, LOW);

  lcd.init();
  lcd.backlight();
  lcd.print("Safe System Act.");
  delay(1500);
  lcd.clear();

  Serial.println("Multi-Zone Fire System Active");
}

void loop() {
  int flameDetected = digitalRead(FLAME_PIN); // active-LOW
  int gasLevel = analogRead(GAS_PIN);

  bool zone1_alert = (flameDetected == LOW);
  bool zone2_alert = (gasLevel > GAS_THRESHOLD);

  Serial.print("Flame (Z1): ");
  Serial.print(zone1_alert ? "ALERT" : "OK");
  Serial.print(" | Gas (Z2): ");
  Serial.print(gasLevel);
  Serial.println(zone2_alert ? " (ALERT)" : " (OK)");

  if (zone1_alert || zone2_alert) {
    digitalWrite(RELAY_PIN, HIGH);  // Cut power/isolate load
    digitalWrite(BUZZER_PIN, HIGH); // Alarm sounded

    lcd.setCursor(0, 0);
    lcd.print("SYSTEM ALARM!   ");

    lcd.setCursor(0, 1);
    if (zone1_alert && zone2_alert) {
      lcd.print("FLAME + GAS DET ");
    } else if (zone1_alert) {
      lcd.print("FLAME ZONE 1    ");
    } else {
      lcd.print("GAS ZONE 2      ");
    }
  } else {
    digitalWrite(RELAY_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);

    lcd.setCursor(0, 0);
    lcd.print("Z1:OK   Z2:OK   ");
    lcd.setCursor(0, 1);
    lcd.print("System Safe     ");
  }

  delay(500);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Flame Sensor**, **MQ-2 Gas Sensor**, **Relay Module**, **Active Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Connect Flame Sensor: **VCC** to **5V**, **DO** to **D2**, **GND** to **GND**.
3. Connect MQ-2 Gas Sensor: **VCC** to **5V**, **AO** to **A1**, **GND** to **GND**.
4. Connect Relay: **VCC** to **5V**, **IN** to **D8**, **GND** to **GND**.
5. Connect Buzzer: **VCC** to **D7**, **GND** to **GND**.
6. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
7. Paste the code, select the interpreted mode runtime, and click **Run**.
8. Modify the gas sensor value using the slider, or toggle the flame sensor pin level, and observe the alarms.

## Expected Output

Terminal:
```
Multi-Zone Fire System Active
Flame (Z1): OK | Gas (Z2): 180 (OK)
Flame (Z1): ALERT | Gas (Z2): 185 (OK)
```

LCD Display:
```
Z1:OK   Z2:OK   
System Safe     
```

## Expected Canvas Behavior
| Flame Pin (D2) | Gas Level (A1) | Buzzer (D7) | Relay (D8) | LCD Line 1 | LCD Line 2 |
| --- | --- | --- | --- | --- | --- |
| HIGH | 150 | LOW | LOW | `Z1:OK   Z2:OK` | `System Safe` |
| LOW | 150 | HIGH | HIGH | `SYSTEM ALARM!` | `FLAME ZONE 1` |
| HIGH | 450 | HIGH | HIGH | `SYSTEM ALARM!` | `GAS ZONE 2` |
| LOW | 450 | HIGH | HIGH | `SYSTEM ALARM!` | `FLAME + GAS DET` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `flameDetected == LOW` | Flame sensors normally pull the output low when infrared fire signatures are detected. |
| `zone1_alert \|\| zone2_alert` | Logical OR structure: triggers alarm if either zone fails threshold verification. |
| `digitalWrite(RELAY_PIN, HIGH)` | Activates the electromagnet to disconnect normally-closed contacts and isolate external loads. |

## Hardware & Safety Concept: Fail-Safe Isolation
In real industrial fire panels, relays default to a configuration where power is supplied on the "Normally Open" (NO) terminal only when actively driven by the controller, or they use a NC (Normally Closed) setup designed to de-energize when alarm circuits trip. This ensures that if the microcontroller itself loses power, the hazard isolation triggers automatically.

## Try This! (Challenges)
1. **Latching Alarm**: Modify the code so that once the alarm is triggered, it remains ON until a reset button (D3) is pressed, even if the flame/gas levels return to normal.
2. **Beep Pattern Differentiation**: Program the buzzer to emit rapid short pulses for flame alerts, and a steady tone for gas alerts.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Gas level never changes | MQ-2 slider value is not aligned with pin A1 | Check that the wire connects to A1 and not A0. |
| Flame never triggers alert | Wrong polarity check | Confirm that the digital read logic checks for `LOW` (active) rather than `HIGH`. |

## Mode Notes
This multi-hazard logic and sequence structure runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [32 - Gas Alarm](../intermediate/32-gas-alarm.md)
- [34 - Flame Alarm](../intermediate/34-flame-alarm.md)
- [88 - Fire Safety System](../advanced/88-fire-safety-system.md)
