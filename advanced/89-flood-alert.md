# 89 - Flood Alert

Monitor water levels using a capacitive soil/water sensor and trigger escalating alert outputs (LED indicator + buzzer + LCD warning) when water rises above safe thresholds.

## Goal
Learn how to implement a multi-level threshold system using a single analog sensor to drive multiple output responses that escalate in severity as the hazard increases.

## What You Will Build
The system reads the water/soil moisture sensor on pin A0 and classifies the level into three zones:
- **Safe (< 300)**: Green LED on, no alarm.
- **Caution (300 to 600)**: Yellow LED on, slow beep every 2 seconds.
- **Flood Alert (> 600)**: Red LED on, rapid siren, LCD shows "FLOOD ALERT!".

**Why A0, D3, D4, D5, D8?** Pin A0 reads the sensor. Pins D3/D4/D5 drive three LEDs. Pin D8 drives the buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Water Level Sensor | `moisture_sensor` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Yellow) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes (per LED) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Water Sensor | VCC | 5V | Power supply |
| Water Sensor | A0 | A0 | Analog level output |
| Water Sensor | GND | GND | Ground reference |
| Green LED | Anode (+) | D3 | Safe zone indicator |
| Green LED | Cathode (-) | GND | Via 220 ohm resistor |
| Yellow LED | Anode (+) | D4 | Caution zone indicator |
| Yellow LED | Cathode (-) | GND | Via 220 ohm resistor |
| Red LED | Anode (+) | D5 | Flood alert indicator |
| Red LED | Cathode (-) | GND | Via 220 ohm resistor |
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

const int WATER_PIN  = A0;
const int GREEN_PIN  = 3;
const int YELLOW_PIN = 4;
const int RED_PIN    = 5;
const int BUZZER_PIN = 8;

const int CAUTION_THRESHOLD = 300;
const int FLOOD_THRESHOLD   = 600;

LiquidCrystal_I2C lcd(0x27, 16, 2);

void allLEDsOff() {
  digitalWrite(GREEN_PIN,  LOW);
  digitalWrite(YELLOW_PIN, LOW);
  digitalWrite(RED_PIN,    LOW);
}

void setup() {
  Serial.begin(9600);
  
  pinMode(GREEN_PIN,  OUTPUT);
  pinMode(YELLOW_PIN, OUTPUT);
  pinMode(RED_PIN,    OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  allLEDsOff();
  noTone(BUZZER_PIN);
  
  lcd.init();
  lcd.backlight();
  lcd.print("Flood Monitor");
  delay(1500);
  lcd.clear();
  
  Serial.println("Flood Alert System Active");
}

void loop() {
  int waterLevel = analogRead(WATER_PIN);
  
  Serial.print("Water Level: "); Serial.println(waterLevel);
  
  allLEDsOff();
  
  if (waterLevel >= FLOOD_THRESHOLD) {
    // Flood alert
    digitalWrite(RED_PIN, HIGH);
    tone(BUZZER_PIN, 1100, 400);
    
    lcd.setCursor(0, 0);
    lcd.print("Water Level:    ");
    lcd.setCursor(0, 1);
    lcd.print("!! FLOOD ALERT!!");
    delay(450);
    
    tone(BUZZER_PIN, 650, 400);
    delay(450);
    
  } else if (waterLevel >= CAUTION_THRESHOLD) {
    // Caution zone
    digitalWrite(YELLOW_PIN, HIGH);
    noTone(BUZZER_PIN);
    
    lcd.setCursor(0, 0);
    lcd.print("Water Level:    ");
    lcd.setCursor(0, 1);
    lcd.print("CAUTION: Rising ");
    
    tone(BUZZER_PIN, 800, 100);
    delay(2000); // Slow single beep every 2 seconds
    
  } else {
    // Safe zone
    digitalWrite(GREEN_PIN, HIGH);
    noTone(BUZZER_PIN);
    
    lcd.setCursor(0, 0);
    lcd.print("Water Level:    ");
    lcd.setCursor(0, 1);
    lcd.print("Status: SAFE    ");
    delay(500);
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Water Level Sensor**, **three LEDs** (Green, Yellow, Red), **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Connect water sensor: **VCC** to **5V**, **A0** to **A0**, **GND** to **GND**.
3. Connect each LED anode to **D3** (Green), **D4** (Yellow), **D5** (Red), cathodes to **GND**.
4. Connect Buzzer: **+** to **D8**, **-** to **GND**.
5. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
6. Paste the code into the editor.
7. Use the default Arduino interpreted runtime.
8. Click **Run**.
9. Adjust the water sensor slider and observe the LED, buzzer, and LCD change states.

## Expected Output

Terminal:
```
Flood Alert System Active
Water Level: 150
Water Level: 420
Water Level: 750
```

## Expected Canvas Behavior

| Sensor Reading | Zone | Active LED | Buzzer | LCD Line 2 |
| --- | --- | --- | --- | --- |
| 0 to 299 | Safe | Green | Silent | `Status: SAFE` |
| 300 to 599 | Caution | Yellow | Single beep / 2s | `CAUTION: Rising` |
| 600 to 1023 | Flood Alert | Red | Rapid siren | `!! FLOOD ALERT!!` |

## Code Walkthrough

| Expression | What It Does |
| --- | --- |
| `allLEDsOff()` | Centralised helper function that turns all three indicator LEDs off before selectively turning one on. Prevents multiple LEDs being active together. |
| `tone(BUZZER_PIN, 1100, 400)` and `tone(BUZZER_PIN, 650, 400)` | Alternates between high and low frequencies to create a distinctive siren pattern. |

## Hardware & Safety Concept: Escalating Alert Design
Good safety system UX uses escalating alerts so operators can assess urgency at a glance without reading text.
- **Green**: No action required.
- **Yellow**: Awareness — take a look but no immediate danger.
- **Red**: Immediate action required.
This is the same color coding convention used in traffic lights, industrial control panels, and hospital patient monitoring systems.

## Try This! (Challenges)
1. **Water Rising Rate**: Store the last two readings (2 seconds apart). If the difference is greater than 50 units, add `(Rising fast!)` to the LCD display.
2. **Daily High Mark**: Track the maximum water level seen since startup using a `maxLevel` variable. Display it on LCD line 1 as `Peak:XXX`.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| All three LEDs on at once | Missing `allLEDsOff()` call before zones | Verify that `allLEDsOff()` is called at the top of every `loop()` cycle. |
| Caution and flood thresholds swapped | Variable values incorrect | Confirm that `CAUTION_THRESHOLD` (300) is less than `FLOOD_THRESHOLD` (600). |

## Mode Notes
These patterns (multi-threshold analog checks, LED switching, tone generation, and LCD output) are supported by MbedO interpreted mode.

## Related Projects
- [88 - Fire Safety System](88-fire-safety-system.md)
- [55 - Distance Alert LED](../intermediate/55-distance-alert-led.md)
- [90 - Smart Thermostat](90-smart-thermostat.md)
