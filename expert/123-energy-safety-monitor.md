# 123 - Energy Safety Monitor

Build an active overcurrent protection system using an ACS712 analog current sensor, a control relay acting as a circuit breaker, a warning buzzer, and a character LCD screen to log status and measurements.

## Goal
Learn how to sample analog sensor values representing raw voltage, map those values into real electrical units (milliamperes), and establish an automatic circuit trip loop that isolates loads when current thresholds are breached.

## What You Will Build
A digital circuit breaker:
- **ACS712 Sensor**: Measures analog voltage shifts corresponding to passing current.
- **Relay Switch**: Acts as the contactor. Normally ON to power the load; trips OFF (isolates) when overcurrent is detected.
- **Buzzer Siren**: Sounds immediately when a trip event occurs.
- **LCD Status Display**: Prints real-time current load in mA and registers `TRIPPED` if the limit is exceeded.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| ACS712 Current Sensor | `acs712` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| ACS712 Sensor | VCC | 5V | Power supply |
| ACS712 Sensor | OUT | A1 | Analog voltage signal |
| ACS712 Sensor | GND | GND | Ground reference |
| Relay Module | VCC | 5V | Power supply |
| Relay Module | IN | D8 | Relay control (LOW = Normal Closed, HIGH = Tripped/Open) |
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

const int CURRENT_PIN = A1;
const int RELAY_PIN   = 8;
const int BUZZER_PIN  = 7;

// Overcurrent trip limit in mA
const int CURRENT_LIMIT = 500;

LiquidCrystal_I2C lcd(0x27, 16, 2);

bool isTripped = false;

void setup() {
  Serial.begin(9600);
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  // Default: Load connected, alarm off
  digitalWrite(RELAY_PIN, LOW); 
  digitalWrite(BUZZER_PIN, LOW);

  lcd.init();
  lcd.backlight();
  lcd.print("Safety Mon. Act.");
  delay(1500);
  lcd.clear();

  Serial.println("Energy Safety Monitor Active");
}

void loop() {
  int sensorValue = analogRead(CURRENT_PIN);
  
  // ACS712 calibration logic:
  // 512 is the zero-current midpoint (2.5V out of 5V).
  // Each step is ~4.9mV. Sensistivity is 185mV/A for 5A model.
  // We approximate: mA = (analogValue - 512) * 26.4
  int currentMA = (sensorValue - 512) * 26;
  if (currentMA < 0) {
    currentMA = 0; // Ignore negative direction for DC
  }

  Serial.print("Sensor ADC: ");  Serial.print(sensorValue);
  Serial.print(" | Current: ");   Serial.print(currentMA);
  Serial.println(" mA");

  // Check for overcurrent breach
  if (currentMA > CURRENT_LIMIT) {
    isTripped = true;
  }

  // Trip action
  if (isTripped) {
    digitalWrite(RELAY_PIN, HIGH);  // Open contactor, cut power
    digitalWrite(BUZZER_PIN, HIGH); // Alarm sounded
    
    lcd.setCursor(0, 0);
    lcd.print("OVERCURRENT TRIP");
    lcd.setCursor(0, 1);
    lcd.print("I: ");
    lcd.print(currentMA);
    lcd.print("mA  RESET   ");
  } else {
    digitalWrite(RELAY_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);

    lcd.setCursor(0, 0);
    lcd.print("System Normal   ");
    lcd.setCursor(0, 1);
    lcd.print("Load: ");
    lcd.print(currentMA);
    lcd.print(" mA     ");
  }

  // Manual reset mechanism via serial console command
  if (Serial.available()) {
    char c = Serial.read();
    if (c == 'r' || c == 'R') {
      isTripped = false;
      Serial.println("System Reset Initiated.");
      lcd.clear();
      lcd.print("Resetting...");
      delay(1000);
      lcd.clear();
    }
  }

  delay(300);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **ACS712 Current Sensor**, **Relay Module**, **Active Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Connect ACS712: **VCC** to **5V**, **OUT** to **A1**, **GND** to **GND**.
3. Connect Relay: **VCC** to **5V**, **IN** to **D8**, **GND** to **GND**.
4. Connect Buzzer: **VCC** to **D7**, **GND** to **GND**.
5. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
6. Paste code, select the interpreted mode, and click **Run**.
7. Adjust the ACS712 analog sensor slider. Move it high (past 532 ADC steps) to trigger the overcurrent trip.
8. Type `r` into the Serial Monitor command bar and click send to reset the breaker.

## Expected Output

Terminal:
```
Energy Safety Monitor Active
Sensor ADC: 512 | Current: 0 mA
Sensor ADC: 535 | Current: 598 mA
System Reset Initiated.
```

LCD Display:
```
OVERCURRENT TRIP
I: 598mA  RESET   
```

## Expected Canvas Behavior
| ACS712 Output (A1) | Load Status (Relay D8) | Alarm (Buzzer D7) | LCD Line 1 | LCD Line 2 |
| --- | --- | --- | --- | --- |
| 512 (2.5V) | LOW (Closed) | LOW | `System Normal` | `Load: 0 mA` |
| 520 | LOW (Closed) | LOW | `System Normal` | `Load: 208 mA` |
| 535 (Tripped) | HIGH (Open) | HIGH | `OVERCURRENT TRIP` | `I: 598mA  RESET` |
| 512 (After trip) | HIGH (Open) | HIGH | `OVERCURRENT TRIP` | `I: 0mA  RESET` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `(sensorValue - 512)` | Calculates the offset from the zero-current midpoint reference (2.5V Vcc bias). |
| `isTripped = true` | Latches the alarm state so that the relay remains open even if the current drops to zero. |
| `Serial.read() == 'r'` | Receives reset command to re-enable closed-circuit operations. |

## Hardware & Safety Concept: Current Calibration
The ACS712 utilizes a Hall effect sensor to measure magnetic field lines generated by passing current, converting this into proportional output voltage. Because it is powered at 5V, 0A of current yields a baseline output voltage of 2.5V (512 ADC steps). Calibration shifts this midpoint to zero and applies scaling factors depending on the sensor's sensitivity rating (e.g. 185mV/A, 100mV/A, or 66mV/A).

## Try This! (Challenges)
1. **Peak Hold Logging**: Program the LCD to display the maximum current value reached since startup on the bottom-right corner.
2. **Current Surge Timer**: Allow the current to exceed the threshold for up to 1.5 seconds before tripping the relay, preventing false trips from temporary current spikes (motor starting surges).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Current reads ~13000mA continuously | Floating input pin | Verify that the ACS712 OUT pin is wired to A1, not an unconfigured digital pin. |
| CONTROLLER doesn't reset | Serial input parsing fail | Confirm line endings in the serial input or ensure you send exactly the letter `r`. |

## Mode Notes
The analog mapping and latching logic run locally and are supported by MbedO interpreted mode.

## Related Projects
- [15 - Analog Meter Serial](../beginner/15-analog-meter-serial.md)
- [40 - Water Pump Control](../intermediate/40-water-pump-control.md)
- [97 - Gas + Relay Safety](../advanced/97-gas-relay-safety.md)
