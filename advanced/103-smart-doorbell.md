# 103 - Smart Doorbell

Detect motion using a PIR sensor and immediately announce "Visitor Detected" on a 16x2 I2C LCD while playing a doorbell chime on a passive buzzer — a complete, self-contained smart doorbell.

## Goal
Learn how to combine a digital motion sensor, a tone-generating buzzer, and an LCD display into a coordinated response sequence triggered by a single sensor event.

## What You Will Build
When the PIR sensor detects motion the LCD immediately shows "Visitor Detected" on line 1 and the time since power-on (in seconds) on line 2. The buzzer plays a two-note doorbell chime (ding-dong). After 3 seconds the LCD returns to its idle message "Awaiting visitor". All events are printed to the Serial Monitor.

**Why these pins?** The PIR digital output goes to `D2`. The buzzer uses `D8` (any digital pin works with `tone()`). The LCD uses the I2C bus on A4/A5.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| PIR Sensor | VCC | 5V | Power supply |
| PIR Sensor | OUT | D2 | Digital motion signal |
| PIR Sensor | GND | GND | Ground reference |
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

const int PIR_PIN    = 2;
const int BUZZER_PIN = 8;

LiquidCrystal_I2C lcd(0x27, 16, 2);

bool showingAlert = false;
int  alertCountdown = 0;

void setup() {
  Serial.begin(9600);
  pinMode(PIR_PIN,    INPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Doorbell");
  lcd.setCursor(0, 1);
  lcd.print("Initialising...");
  delay(2000);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Awaiting visitor");

  Serial.println("Smart Doorbell Ready");
}

void loop() {
  int motion = digitalRead(PIR_PIN);

  if (motion == HIGH && !showingAlert) {
    showingAlert   = true;
    alertCountdown = 6; // 6 x 500 ms = 3 seconds

    // Play doorbell: ding
    tone(BUZZER_PIN, 1047, 400);
    delay(450);
    // dong
    tone(BUZZER_PIN, 784, 600);
    delay(650);
    noTone(BUZZER_PIN);

    long seconds = millis() / 1000;

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Visitor Detected");
    lcd.setCursor(0, 1);
    lcd.print("At: ");
    lcd.print(seconds);
    lcd.print("s      ");

    Serial.print("Visitor Detected at ");
    Serial.print(seconds);
    Serial.println("s");
  }

  if (showingAlert) {
    alertCountdown--;
    if (alertCountdown <= 0) {
      showingAlert = false;
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Awaiting visitor");
      Serial.println("Doorbell idle");
    }
  }

  delay(500);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **PIR Motion Sensor**, **Passive Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Connect PIR: **VCC** to **5V**, **OUT** to **D2**, **GND** to **GND**.
3. Connect Buzzer: **+** to **D8**, **−** to **GND**.
4. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.
8. Double-click the PIR sensor on the canvas to toggle motion — observe the LCD message change and the buzzer play the chime.

## Expected Output

Terminal:
```
Smart Doorbell Ready
Visitor Detected at 12s
Doorbell idle
```

LCD Display:
```
Visitor Detected
At: 12s
```

## Expected Canvas Behavior
| PIR State | LCD Line 1 | LCD Line 2 | Buzzer |
| --- | --- | --- | --- |
| LOW (no motion) | `Awaiting visitor` | (blank) | Silent |
| HIGH (motion) | `Visitor Detected` | `At: 12s` | Ding-dong chime |
| After 3 s (reset) | `Awaiting visitor` | (blank) | Silent |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `motion == HIGH && !showingAlert` | Triggers the doorbell only on a fresh rising edge — prevents re-triggering while the alert is still displayed. |
| `tone(BUZZER_PIN, 1047, 400)` | Plays the musical note C6 (1047 Hz) for 400 ms, forming the "ding" of the chime. |
| `tone(BUZZER_PIN, 784, 600)` | Plays the note G5 (784 Hz) for 600 ms, forming the "dong" of the chime. |
| `millis() / 1000` | Converts the running millisecond counter to seconds for a human-readable timestamp. |
| `alertCountdown--` | Counts down each 500 ms loop iteration; when zero the LCD resets to idle. |

## Hardware & Safety Concept: Passive vs Active Buzzers
A **passive buzzer** is simply a piezoelectric disc — it makes no sound on its own and requires an oscillating signal at the desired frequency. `tone()` generates this oscillating signal.
An **active buzzer** contains an internal oscillator and beeps at a fixed frequency whenever power is applied — you cannot change its pitch.
- Always use a **passive buzzer** when you need musical tones or variable frequencies.
- The `tone()` function uses Timer 2 on the Arduino Uno; this shares hardware with PWM on pins 3 and 11, which may cause interference if those pins are simultaneously used for `analogWrite`.

## Try This! (Challenges)
1. **Visitor Count**: Increment an integer counter each time motion is detected and display `"Visitors: N"` on line 2 during the idle state so you can see how many times the bell has rung since power-on.
2. **Custom Melody**: Change the two `tone()` frequencies to mimic a different bell pattern, such as a Westminster chime using notes E5 (659 Hz), D5 (587 Hz), C5 (523 Hz), and G4 (392 Hz).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Doorbell triggers repeatedly | PIR stays HIGH | Add the `!showingAlert` guard (already in code); ensure PIR sensitivity is not set too high. |
| Buzzer is silent | Active buzzer used | Replace with a passive buzzer or piezo disc — active buzzers do not respond to `tone()`. |
| LCD stays blank | Wrong I2C address | Try changing `0x27` to `0x3F` in the constructor. |
| Timestamp shows 0 | `millis()` cast issue | Use `long seconds = millis() / 1000` to avoid 16-bit overflow in the division. |

## Mode Notes
All patterns used here (`digitalRead`, `tone()`, `noTone()`, `lcd.print`, `if/else`) are fully supported by MbedO interpreted mode.

## Related Projects
- [102 - BT Home Controller](102-bt-home-controller.md)
- [104 - Water Level Station](104-water-level-station.md)
- [57 - PIR Motion Alert](../intermediate/57-pir-motion-alert.md)
- [66 - Buzzer Melody](../intermediate/66-buzzer-melody.md)
