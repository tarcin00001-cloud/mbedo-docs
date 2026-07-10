# 98 - Proximity Guard

Use an HC-SR04 ultrasonic sensor as a proximity tripwire: when an object enters a defined guard zone (closer than a threshold distance), the system triggers a buzzer alarm and displays the intruder distance on an LCD.

## Goal
Learn how to turn a distance sensor into a proximity security tripwire using range thresholds, with configurable guard-zone distance and clear-zone recovery logic.

## What You Will Build
The system continuously measures distance in front of the HC-SR04 sensor. Three zones are defined:
- **Clear (> 80 cm)**: LCD shows "Area: CLEAR", Green LED on.
- **Approach (30 to 80 cm)**: LCD shows "Approach: XXcm", Yellow LED on, slow beep.
- **Breach (< 30 cm)**: LCD shows "BREACH: XXcm!!", Red LED on, rapid siren, relay activates.

**Why D2, D3, D4, D5, D6, D7, D8?** Pins D2/D3 are TRIG/ECHO. Pins D4/D5/D6 drive LEDs. Pin D7 controls the relay. Pin D8 drives the buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hc_sr04` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Yellow) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |
| 5V Relay | `relay` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes (per LED) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 | VCC | 5V | Power supply |
| HC-SR04 | TRIG | D2 | Trigger pulse |
| HC-SR04 | ECHO | D3 | Echo return |
| HC-SR04 | GND | GND | Ground reference |
| Green LED | Anode (+) | D4 | Clear zone indicator |
| Green LED | Cathode (-) | GND | Via 220 ohm resistor |
| Yellow LED | Anode (+) | D5 | Approach indicator |
| Yellow LED | Cathode (-) | GND | Via 220 ohm resistor |
| Red LED | Anode (+) | D6 | Breach indicator |
| Red LED | Cathode (-) | GND | Via 220 ohm resistor |
| Relay | IN | D7 | Control signal |
| Relay | VCC | 5V | Power supply |
| Relay | GND | GND | Ground reference |
| Buzzer | + | D8 | Signal pin |
| Buzzer | - | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int TRIG_PIN  = 2;
const int ECHO_PIN  = 3;
const int GREEN_PIN = 4;
const int YEL_PIN   = 5;
const int RED_PIN   = 6;
const int RELAY_PIN = 7;
const int BUZ_PIN   = 8;

// Guard zone thresholds in centimeters
const int BREACH_CM   = 30;
const int APPROACH_CM = 80;

LiquidCrystal_I2C lcd(0x27, 16, 2);

long readDistanceCM() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  long duration = pulseIn(ECHO_PIN, HIGH);
  return duration * 0.034 / 2;
}

void allLEDsOff() {
  digitalWrite(GREEN_PIN, LOW);
  digitalWrite(YEL_PIN,   LOW);
  digitalWrite(RED_PIN,   LOW);
}

void setup() {
  Serial.begin(9600);
  
  pinMode(TRIG_PIN,  OUTPUT);
  pinMode(ECHO_PIN,  INPUT);
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(YEL_PIN,   OUTPUT);
  pinMode(RED_PIN,   OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZ_PIN,   OUTPUT);
  
  allLEDsOff();
  digitalWrite(RELAY_PIN, LOW);
  noTone(BUZ_PIN);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0); lcd.print("Proximity Guard ");
  lcd.setCursor(0, 1); lcd.print("Starting...     ");
  delay(1500);
  lcd.clear();
  
  Serial.println("Proximity Guard System Active");
  Serial.print("Guard zones: Breach < "); Serial.print(BREACH_CM);
  Serial.print(" cm | Approach < "); Serial.print(APPROACH_CM); Serial.println(" cm");
}

void loop() {
  long dist = readDistanceCM();
  
  if (dist <= 0 || dist > 400) {
    Serial.println("Sensor out of range.");
    delay(200);
    return;
  }
  
  Serial.print("Distance: "); Serial.print(dist); Serial.println(" cm");
  
  allLEDsOff();
  
  if (dist < BREACH_CM) {
    // BREACH zone
    digitalWrite(RED_PIN,   HIGH);
    digitalWrite(RELAY_PIN, HIGH);
    
    lcd.setCursor(0, 0); lcd.print("!! BREACH !!    ");
    lcd.setCursor(0, 1);
    lcd.print("Dist:");
    lcd.print(dist);
    lcd.print("cm      ");
    
    tone(BUZ_PIN, 1100, 250);
    delay(300);
    tone(BUZ_PIN, 700, 250);
    delay(300);
    
  } else if (dist < APPROACH_CM) {
    // APPROACH zone
    digitalWrite(YEL_PIN,   HIGH);
    digitalWrite(RELAY_PIN, LOW);
    
    lcd.setCursor(0, 0); lcd.print("Approach: ");
    lcd.print(dist); lcd.print("cm  ");
    lcd.setCursor(0, 1); lcd.print("Stay back!      ");
    
    tone(BUZ_PIN, 800, 100);
    delay(1500);
    
  } else {
    // CLEAR zone
    digitalWrite(GREEN_PIN, HIGH);
    digitalWrite(RELAY_PIN, LOW);
    noTone(BUZ_PIN);
    
    lcd.setCursor(0, 0); lcd.print("Area: CLEAR     ");
    lcd.setCursor(0, 1); lcd.print("Dist:");
    lcd.print(dist); lcd.print("cm      ");
    delay(300);
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-SR04 Sensor**, **three LEDs** (G/Y/R), **Relay**, **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Connect HC-SR04: **TRIG** to **D2**, **ECHO** to **D3**, **VCC** to **5V**, **GND** to **GND**.
3. Connect LEDs: **D4** (Green), **D5** (Yellow), **D6** (Red).
4. Connect Relay: **IN** to **D7**.
5. Connect Buzzer: **+** to **D8**.
6. Connect LCD: **SDA** to **A4**, **SCL** to **A5**.
7. Paste the code into the editor.
8. Use the default Arduino interpreted runtime.
9. Click **Run**.
10. Adjust the HC-SR04 distance slider from far (> 80 cm) down to near (< 30 cm) and observe zone transitions.

## Expected Output

Terminal:
```
Proximity Guard System Active
Guard zones: Breach < 30 cm | Approach < 80 cm
Distance: 120 cm
Distance: 55 cm
Distance: 18 cm
```

## Expected Canvas Behavior

| HC-SR04 Distance | Zone | Relay | Active LED | Buzzer | LCD Line 1 |
| --- | --- | --- | --- | --- | --- |
| > 80 cm | Clear | Open | Green | Silent | `Area: CLEAR` |
| 30 to 80 cm | Approach | Open | Yellow | Slow beep | `Approach: XXcm` |
| < 30 cm | Breach | Closed | Red | Rapid siren | `!! BREACH !!` |

## Code Walkthrough

| Expression | What It Does |
| --- | --- |
| `readDistanceCM()` helper | Encapsulates the full TRIG/ECHO pulse sequence and calculation into a single reusable function call. |
| `dist <= 0 \|\| dist > 400` guard | Filters out invalid readings (sensor out of range, no echo) to prevent false alarms from measurement noise. |

## Hardware & Safety Concept: Ultrasonic Dead Zones
The HC-SR04 has a **blind spot** (dead zone) of approximately 2 to 4 cm directly in front of the sensor. Objects closer than 2 cm may not be detected at all because the transmitter pulse overlaps with the receiver window.
- In a physical deployment, mount the sensor so that even the closest expected object remains beyond 5 cm from the face.
- In the canvas simulation, readings below 5 cm may return 0. The `dist <= 0` guard in the code handles this case.

## Try This! (Challenges)
1. **Adjustable Zones**: Wire two potentiometers to A0 and A1 to set the APPROACH and BREACH thresholds dynamically without recompiling.
2. **Event Duration Logging**: Record how long an object remained in the BREACH zone (using `millis()` start/stop timing). Print the duration in seconds to the Terminal when the object moves away.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Distance always shows 0 | TRIG/ECHO pins swapped | Verify TRIG connects to D2 (OUTPUT) and ECHO to D3 (INPUT). |
| Breach zone never triggers | Threshold too low | Lower `BREACH_CM` to 50 cm initially to verify the detection pipeline works. |

## Mode Notes
These patterns (ultrasonic pulse timing, zone thresholds, relay, and LCD output) are supported by MbedO interpreted mode.

## Related Projects
- [56 - Parking Beeper](../intermediate/56-parking-beeper.md)
- [89 - Flood Alert](89-flood-alert.md)
- [96 - PIR Zone Alert](96-pir-zone-alert.md)
