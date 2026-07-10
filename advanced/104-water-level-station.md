# 104 - Water Level Station

Read an analog water level sensor, display the percentage on a 16x2 I2C LCD, and trigger a buzzer alarm when the water exceeds a critical threshold — simulating a tank overflow or flood warning system.

## Goal
Learn how to convert a raw analog sensor reading into a percentage using `map()`, display it on an LCD, and implement a threshold-based alarm using `tone()`.

## What You Will Build
Every 500 ms the sketch reads the water level sensor on `A0` and converts the raw 0–1023 value to a 0–100% scale. The percentage and a status label are shown on the LCD. If the level exceeds 80% the buzzer sounds an alarm and the LCD shows "FLOOD ALERT!". Below the threshold the buzzer is silent and the LCD shows the level normally.

**Why these pins?** The water level sensor outputs an analog voltage proportional to how much of its sensing strips are submerged and connects to `A0`. The buzzer goes to `D8`. The LCD uses the I2C bus on A4/A5.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Water Level Sensor | `water_level` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Water Level Sensor | VCC | 5V | Power supply |
| Water Level Sensor | GND | GND | Ground reference |
| Water Level Sensor | SIG | A0 | Analog signal output |
| Passive Buzzer | + | D8 | PWM tone output |
| Passive Buzzer | − | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data |
| 16x2 I2C LCD | SCL | A5 | I2C Clock |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int WATER_PIN  = A0;
const int BUZZER_PIN = 8;
const int FLOOD_THRESHOLD = 80; // Percent

LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  pinMode(BUZZER_PIN, OUTPUT);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Water Level Stn");
  lcd.setCursor(0, 1);
  lcd.print("Initialising...");
  delay(1500);
  lcd.clear();

  Serial.println("Water Level Station Ready");
}

void loop() {
  int raw     = analogRead(WATER_PIN);
  int percent = map(raw, 0, 1023, 0, 100);

  Serial.print("Raw: ");
  Serial.print(raw);
  Serial.print(" | Level: ");
  Serial.print(percent);
  Serial.println("%");

  lcd.setCursor(0, 0);
  lcd.print("Level: ");
  lcd.print(percent);
  lcd.print("%   ");

  if (percent >= FLOOD_THRESHOLD) {
    tone(BUZZER_PIN, 880);
    lcd.setCursor(0, 1);
    lcd.print("!! FLOOD ALERT !!");
    Serial.println("*** FLOOD ALERT ***");
  } else if (percent >= 50) {
    noTone(BUZZER_PIN);
    lcd.setCursor(0, 1);
    lcd.print("Status: HIGH    ");
  } else if (percent >= 20) {
    noTone(BUZZER_PIN);
    lcd.setCursor(0, 1);
    lcd.print("Status: NORMAL  ");
  } else {
    noTone(BUZZER_PIN);
    lcd.setCursor(0, 1);
    lcd.print("Status: LOW     ");
  }

  delay(500);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Water Level Sensor**, **Passive Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Connect Water Level Sensor: **VCC** to **5V**, **GND** to **GND**, **SIG** to **A0**.
3. Connect Buzzer: **+** to **D8**, **−** to **GND**.
4. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.
8. Double-click the Water Level Sensor on the canvas and drag the slider — watch the LCD percentage and buzzer change as you cross the thresholds.

## Expected Output

Terminal:
```
Water Level Station Ready
Raw: 512 | Level: 50%
Raw: 860 | Level: 84%
*** FLOOD ALERT ***
```

LCD Display:
```
Level: 50%
Status: HIGH
```

## Expected Canvas Behavior
| Sensor Raw | Percent | LCD Line 1 | LCD Line 2 | Buzzer |
| --- | --- | --- | --- | --- |
| 0–203 | 0–19% | `Level: N%` | `Status: LOW` | Off |
| 204–511 | 20–49% | `Level: N%` | `Status: NORMAL` | Off |
| 512–818 | 50–79% | `Level: N%` | `Status: HIGH` | Off |
| 819–1023 | 80–100% | `Level: N%` | `!! FLOOD ALERT !!` | Continuous 880 Hz |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `map(raw, 0, 1023, 0, 100)` | Linearly converts the ADC value (0–1023) to a percentage (0–100). |
| `percent >= FLOOD_THRESHOLD` | Compares the mapped percentage against the 80% alarm threshold. |
| `tone(BUZZER_PIN, 880)` | Starts a continuous 880 Hz (A5 musical note) alarm tone with no duration limit. |
| `noTone(BUZZER_PIN)` | Stops any active tone — called in every non-alarm branch to silence the buzzer. |
| Trailing `"    "` spaces | Clears any stale characters on the LCD when a shorter string replaces a longer one. |

## Hardware & Safety Concept: Analog Sensors and ADC Resolution
The Arduino Uno's ADC (Analog-to-Digital Converter) has 10-bit resolution, producing values from 0 to 1023. Each step represents approximately 4.9 mV (5 V ÷ 1024).
- Water level sensors use parallel conducting strips; more strips submerged means lower resistance and therefore higher voltage at the SIG pin.
- Over time, electrolysis corrodes the sensor strips if left submerged continuously — power the sensor from a digital output pin and only set it HIGH just before reading.
- Never use a water level sensor with AC mains circuits; always use battery or USB power for student projects.

## Try This! (Challenges)
1. **Power-Save Trick**: Wire the sensor VCC to a digital output pin (e.g., `D4`) and call `digitalWrite(4, HIGH)` only for 10 ms before each `analogRead`, then set it LOW. This greatly extends the sensor's lifespan by reducing electrolysis.
2. **Percentage Bar**: Use custom LCD characters (`lcd.createChar`) to draw a simple bar-graph of the water level across the bottom row, one filled block per 10%.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Always reads 0% | Sensor not powered | Confirm 5V and GND are connected; check SIG goes to A0 not a digital pin. |
| Always reads 100% | SIG shorted to VCC | Inspect wiring; confirm SIG is the middle pin and not VCC. |
| Buzzer does not stop | `noTone()` not reached | Ensure the `else if` and `else` branches each call `noTone(BUZZER_PIN)`. |
| LCD shows garbled text | Trailing spaces missing | Add enough trailing spaces after each `lcd.print` call to overwrite previous characters. |

## Mode Notes
All patterns used here (`analogRead`, `map()`, `tone()`, `noTone()`, `lcd.print`, `if/else if/else`) are fully supported by MbedO interpreted mode.

## Related Projects
- [103 - Smart Doorbell](103-smart-doorbell.md)
- [105 - Tilt Alarm](105-tilt-alarm.md)
- [54 - Water Level Alert](../intermediate/54-water-level-alert.md)
- [86 - Weather Station](86-weather-station.md)
