# 133 - Pico Car Parking

Build a smart parking lot slot tracker that monitors two spaces using ultrasonic rangefinders and displays slot availability on an LCD.

## Goal
Learn how to manage two separate high-speed pulse sensors (HC-SR04) in parallel, control occupancy LEDs, and update a numeric slot tracker on an LCD.

## What You Will Build
A smart parking space tracking system:
- **Ultrasonic Sensor 1 (Trig GP14, Echo GP15)**: Monitors Slot 1.
- **Ultrasonic Sensor 2 (Trig GP16, Echo GP17)**: Monitors Slot 2.
- **Green/Red LED 1 (GP10)**: Lights RED when Slot 1 is occupied (< 20 cm), and GREEN when empty.
- **Green/Red LED 2 (GP11)**: Lights RED when Slot 2 is occupied, and GREEN when empty.
- **16x2 I2C LCD (GP4, GP5)**: Displays the total number of free slots (e.g. "Free Slots: 2").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensors | `ultrasonic` | Yes (two sensors) | Yes |
| LEDs (Green & Red) | `led` | Yes (represented by separate LEDs) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| Resistors | `resistor` | Optional | Yes (voltage dividers) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 #1 | Trig / Echo | GP14 / GP15 | Slot 1 distance |
| HC-SR04 #2 | Trig / Echo | GP16 / GP17 | Slot 2 distance (with divider) |
| Slot 1 Indicator | Anode | GP10 | RED LED (ON = Occupied) |
| Slot 2 Indicator | Anode | GP11 | RED LED (ON = Occupied) |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int TRIG1 = 14; const int ECHO1 = 15;
const int TRIG2 = 16; const int ECHO2 = 17;
const int RED1  = 10; const int RED2  = 11;

LiquidCrystal_I2C lcd(0x27, 16, 2);

const int OCCUPIED_LIMIT = 20; // Slot limit in cm

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(TRIG1, OUTPUT); pinMode(ECHO1, INPUT);
  pinMode(TRIG2, OUTPUT); pinMode(ECHO2, INPUT);
  pinMode(RED1, OUTPUT);  pinMode(RED2, OUTPUT);

  digitalWrite(TRIG1, LOW);
  digitalWrite(TRIG2, LOW);
  digitalWrite(RED1, LOW);
  digitalWrite(RED2, LOW);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Parking Tracker");
  lcd.setCursor(0, 1);
  lcd.print("Free Slots: --  ");
  delay(1000);
}

void loop() {
  // 1. Measure Slot 1 Distance
  digitalWrite(TRIG1, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG1, LOW);
  long dur1 = pulseIn(ECHO1, HIGH);
  int dist1 = dur1 * 0.0343 / 2;

  delay(60); // Settle delay before scanning sensor 2

  // 2. Measure Slot 2 Distance
  digitalWrite(TRIG2, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG2, LOW);
  long dur2 = pulseIn(ECHO2, HIGH);
  int dist2 = dur2 * 0.0343 / 2;

  // Evaluate occupancy
  bool s1_occupied = (dist1 > 0 && dist1 < OCCUPIED_LIMIT);
  bool s2_occupied = (dist2 > 0 && dist2 < OCCUPIED_LIMIT);

  // Update LEDs
  if (s1_occupied) { digitalWrite(RED1, HIGH); } else { digitalWrite(RED1, LOW); }
  if (s2_occupied) { digitalWrite(RED2, HIGH); } else { digitalWrite(RED2, LOW); }

  // Calculate free slots count
  int freeCount = 2;
  if (s1_occupied) { freeCount = freeCount - 1; }
  if (s2_occupied) { freeCount = freeCount - 1; }

  // Update LCD Display
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Slot 1: ");
  if (s1_occupied) { lcd.print("OCCUPIED"); } else { lcd.print("EMPTY   "); }

  lcd.setCursor(0, 1);
  lcd.print("Free Slots: ");
  lcd.print(freeCount);

  delay(400); // Update twice per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two HC-SR04 Sensors**, **two Red LEDs**, and **I2C LCD** onto the canvas.
2. Connect HC-SR04 #1 to **GP14/GP15**, HC-SR04 #2 to **GP16/GP17**, LEDs to **GP10/GP11**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the distance sliders on both sensors to simulate parking cars and watch the LEDs and LCD change.

## Expected Output

Terminal:
```
Simulation active. Parking slot tracking system online.
```

## Expected Canvas Behavior
* Both Clear (> 30 cm): LED 1 and LED 2 are OFF. LCD reads `Free Slots: 2`.
* Car enters Slot 1 (Distance 10 cm): LED 1 turns ON (Red). LCD reads `Slot 1: OCCUPIED` / `Free Slots: 1`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `freeCount = freeCount - 1` | Decrements the available slot counter if the respective space is occupied. |

## Hardware & Safety Concept: Cross-Talk Prevention
When using multiple ultrasonic sensors close together, waves from Sensor 1 can bounce off targets and trigger the receiver of Sensor 2, causing false readings. To prevent this **cross-talk**, trigger the sensors sequentially with a delay (e.g. `delay(60)`) to let the sound waves from the first scan dissipate before starting the next.

## Try This! (Challenges)
1. **Full Alert Chime**: Connect a buzzer on GP12 and sound a long beep when the parking lot becomes completely full.
2. **Reverse LED Indicator**: Wire two Green LEDs on GP8/GP9 to light up when slots are empty.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Slots show occupied when empty | Sensor range reflection | Verify the sensors are mounted pointing straight down or at the parking floor, and adjust the threshold `OCCUPIED_LIMIT` if the floor is too close. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [75 - Pico Ultrasonic Serial](../intermediate/75-pico-ultrasonic-serial.md)
- [77 - Pico Ultrasonic LCD](../intermediate/77-pico-ultrasonic-lcd.md)
- [118 - Pico Ultrasonic OLED](../intermediate/118-pico-ultrasonic-oled.md)
