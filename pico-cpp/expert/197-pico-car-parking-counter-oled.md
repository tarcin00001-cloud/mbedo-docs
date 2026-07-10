# 197 - Pico Car Parking Counter OLED

Build a smart parking garage manager that monitors entry and exit spaces using rangefinders, controls access servo gates, and displays a slot counter on an OLED.

## Goal
Learn how to manage multiple servo motors and high-speed pulse sensors (HC-SR04) in parallel, implement entry/exit counter rules, and update OLED screens.

## What You Will Build
An automated parking gate manager with graphical dashboard:
- **Entry Ultrasonic Sensor (Trig GP14, Echo GP15)**: Detects arriving cars.
- **Exit Ultrasonic Sensor (Trig GP16, Echo GP17)**: Detects departing cars.
- **Entry Gate Servo (GP10)**: Opens when a car arrives and slots are available.
- **Exit Gate Servo (GP11)**: Opens when a departing car approaches.
- **SSD1306 OLED (GP4, GP5)**: Displays the active count of filled and empty slots, along with a visual occupancy bar.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensors | `ultrasonic` | Yes (two sensors) | Yes |
| Servo Motors | `servo` | Yes (two servos) | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Entry Sensor | Trig / Echo | GP14 / GP15 | Arriving car sensor |
| Exit Sensor | Trig / Echo | GP16 / GP17 | Departing car sensor |
| Entry Gate | Signal | GP10 | Entry barrier servo |
| Exit Gate | Signal | GP11 | Exit barrier servo |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Servo.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int TRIG1 = 14; const int ECHO1 = 15; // Entry sensor
const int TRIG2 = 16; const int ECHO2 = 17; // Exit sensor
const int SRV1  = 10; const int SRV2  = 11; // Entry/Exit gates

Servo entryServo;
Servo exitServo;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

const int TOTAL_SLOTS = 5;
int occupiedSlots = 0;

const int DETECT_LIMIT = 15; // Detection range in cm

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(TRIG1, OUTPUT); pinMode(ECHO1, INPUT);
  pinMode(TRIG2, OUTPUT); pinMode(ECHO2, INPUT);

  entryServo.attach(SRV1);
  exitServo.attach(SRV2);

  // Close both gates initially (0 degrees)
  entryServo.write(0);
  exitServo.write(0);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  updateDisplay();
}

void loop() {
  // 1. Measure Entry distance
  digitalWrite(TRIG1, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG1, LOW);
  long dur1 = pulseIn(ECHO1, HIGH);
  int dist1 = dur1 * 0.0343 / 2;

  delay(60); // Settle delay

  // 2. Measure Exit distance
  digitalWrite(TRIG2, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG2, LOW);
  long dur2 = pulseIn(ECHO2, HIGH);
  int dist2 = dur2 * 0.0343 / 2;

  // 3. Process Entry Gate
  if (dist1 > 0 && dist1 < DETECT_LIMIT) {
    if (occupiedSlots < TOTAL_SLOTS) {
      display.clearDisplay();
      display.setTextColor(SSD1306_WHITE);
      display.setTextSize(1);
      display.setCursor(10, 20);
      display.print("Welcome!");
      display.setCursor(10, 35);
      display.print("Gate Opening... ");
      display.display();
      
      entryServo.write(90); // Open entry gate
      occupiedSlots = occupiedSlots + 1;
      delay(3000);          // Hold gate open
      entryServo.write(0);  // Close entry gate
      updateDisplay();
    } else {
      display.clearDisplay();
      display.setTextColor(SSD1306_WHITE);
      display.setTextSize(1);
      display.setCursor(10, 20);
      display.print("Lot is FULL!");
      display.setCursor(10, 35);
      display.print("Wait for exit...");
      display.display();
      delay(2000);
      updateDisplay();
    }
  }

  // 4. Process Exit Gate
  if (dist2 > 0 && dist2 < DETECT_LIMIT) {
    if (occupiedSlots > 0) {
      display.clearDisplay();
      display.setTextColor(SSD1306_WHITE);
      display.setTextSize(1);
      display.setCursor(10, 20);
      display.print("Thank You!");
      display.setCursor(10, 35);
      display.print("Gate Opening... ");
      display.display();
      
      exitServo.write(90); // Open exit gate
      occupiedSlots = occupiedSlots - 1;
      delay(3000);         // Hold gate open
      exitServo.write(0);  // Close exit gate
      updateDisplay();
    }
  }

  delay(200);
}

void updateDisplay() {
  // Map filled slots to progress bar width (0-128 pixels)
  int barWidth = occupiedSlots * 128 / TOTAL_SLOTS;

  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(22, 3);
  display.print("PARKING MANAGER");

  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);

  // Row 1: Occupied slots count
  display.setCursor(10, 20);
  display.print("Filled: ");
  display.print(occupiedSlots);
  display.print(" / ");
  display.print(TOTAL_SLOTS);

  // Row 2: Free slots count
  display.setCursor(10, 32);
  display.print("Free  : ");
  display.print(TOTAL_SLOTS - occupiedSlots);

  // Row 3: Draw occupancy progress bar
  display.drawRect(0, 46, 128, 14, SSD1306_WHITE);
  if (barWidth > 0) {
    display.fillRect(2, 48, barWidth - 4, 10, SSD1306_WHITE);
  }

  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two HC-SR04 Sensors**, **two Servos**, and **SSD1306 OLED** onto the canvas.
2. Connect HC-SR04 #1 to **GP14/GP15**, HC-SR04 #2 to **GP16/GP17**, Servos to **GP10/GP11**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the entry sensor slider below 15 cm to open the gate and watch the counter and progress bar update on the OLED.

## Expected Output

Terminal:
```
Simulation active. Smart parking gate counter engine online.
```

## Expected Canvas Behavior
* Startup: OLED reads `Filled: 0/5` / `Free: 5`. Gates are at 0°.
* Car arrives at entry sensor (dist1 = 10 cm): OLED reads `Welcome!`, Entry Servo rotates to 90° for 3 seconds, then returns to 0°. OLED updates to `Filled: 1/5` and draws a 20% filled bar.
* Car arrives at exit sensor (dist2 = 10 cm): OLED reads `Thank You!`, Exit Servo rotates to 90° for 3 seconds, then returns to 0°. OLED updates to `Filled: 0/5` and clears the bar.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `occupiedSlots * 128 / TOTAL_SLOTS` | Calculates the width of the graphical occupancy bar based on the ratio of filled slots. |

## Hardware & Safety Concept: Entry/Exit Safety Interlocks
Automated parking gates use safety interlocks (typically ground induction loops or secondary photo-eye beams) directly under the gate. The gate servo will not close as long as these sensors detect a vehicle block, preventing the heavy barrier arm from dropping onto a car.

## Try This! (Challenges)
1. **Full Light Indicator**: Wire a Red LED on GP12 to light up when the lot is completely full, and a Green LED on GP13 when spaces are available.
2. **Audio Warning Beep**: Connect a buzzer on GP9 and sound a warning beep every time a gate opens.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Gate opens but doesn't close | Sensor bounce | Verify that you have a delay (e.g. `delay(3000)`) after opening the gate to allow the car to fully pass before the gate closes. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [77 - Pico Ultrasonic LCD](../intermediate/77-pico-ultrasonic-lcd.md)
- [133 - Pico Car Parking](../advanced/133-pico-car-parking.md)
- [163 - Pico Car Parking Counter](../advanced/163-pico-car-parking-counter.md)
- [189 - Pico Car Parking Counter Logger](189-pico-car-parking-counter-logger.md)
