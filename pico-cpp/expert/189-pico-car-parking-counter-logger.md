# 189 - Pico Car Parking Counter Logger

Build a smart parking garage manager that monitors entry and exit spaces using rangefinders, controls access servo gates, displays slots on an LCD, and streams occupancy logs over Bluetooth.

## Goal
Learn how to manage multiple servo motors and high-speed pulse sensors (HC-SR04) in parallel, implement entry/exit counter rules, update LCD readouts, and stream Bluetooth telemetry reports.

## What You Will Build
An automated parking gate manager with wireless logging:
- **Entry Ultrasonic Sensor (Trig GP14, Echo GP15)**: Detects arriving cars.
- **Exit Ultrasonic Sensor (Trig GP16, Echo GP17)**: Detects departing cars.
- **Entry Gate Servo (GP10)**: Opens when a car arrives and slots are available.
- **Exit Gate Servo (GP11)**: Opens when a departing car approaches.
- **16x2 I2C LCD (GP4, GP5)**: Displays the active count of filled and empty slots.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits occupancy logs wirelessly (e.g., `Occupied,Free`).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensors | `ultrasonic` | Yes (two sensors) | Yes |
| Servo Motors | `servo` | Yes (two servos) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Entry Sensor | Trig / Echo | GP14 / GP15 | Arriving car sensor |
| Exit Sensor | Trig / Echo | GP16 / GP17 | Departing car sensor |
| Entry Gate | Signal | GP10 | Entry barrier servo |
| Exit Gate | Signal | GP11 | Exit barrier servo |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Servo.h>
#include <LiquidCrystal_I2C.h>

const int TRIG1 = 14; const int ECHO1 = 15; // Entry sensor
const int TRIG2 = 16; const int ECHO2 = 17; // Exit sensor
const int SRV1  = 10; const int SRV2  = 11; // Entry/Exit gates

Servo entryServo;
Servo exitServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

const int TOTAL_SLOTS = 5;
int occupiedSlots = 0;

const int DETECT_LIMIT = 15; // Detection range in cm

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART

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

  lcd.init();
  lcd.backlight();
  updateDisplay();

  // Print CSV Header to Bluetooth
  Serial1.println("Occupied,Free");
  delay(1000);
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

  bool countChanged = false;

  // 3. Process Entry Gate
  if (dist1 > 0 && dist1 < DETECT_LIMIT) {
    if (occupiedSlots < TOTAL_SLOTS) {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Welcome!");
      lcd.setCursor(0, 1);
      lcd.print("Gate Opening... ");
      
      entryServo.write(90); // Open entry gate
      occupiedSlots = occupiedSlots + 1;
      countChanged = true;
      delay(3000);          // Hold gate open
      entryServo.write(0);  // Close entry gate
      updateDisplay();
    } else {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Lot is FULL!");
      lcd.setCursor(0, 1);
      lcd.print("Wait for exit...");
      delay(2000);
      updateDisplay();
    }
  }

  // 4. Process Exit Gate
  if (dist2 > 0 && dist2 < DETECT_LIMIT) {
    if (occupiedSlots > 0) {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Thank You!");
      lcd.setCursor(0, 1);
      lcd.print("Gate Opening... ");
      
      exitServo.write(90); // Open exit gate
      occupiedSlots = occupiedSlots - 1;
      countChanged = true;
      delay(3000);         // Hold gate open
      exitServo.write(0);  // Close exit gate
      updateDisplay();
    }
  }

  // 5. Stream CSV logs over Bluetooth if count changes
  if (countChanged) {
    Serial1.print(occupiedSlots);
    Serial1.print(",");
    Serial1.println(TOTAL_SLOTS - occupiedSlots);
  }

  delay(200);
}

void updateDisplay() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Parking Lot Node");
  
  lcd.setCursor(0, 1);
  lcd.print("Filled: ");
  lcd.print(occupiedSlots);
  lcd.print("/");
  lcd.print(TOTAL_SLOTS);
  
  lcd.print(" Free: ");
  lcd.print(TOTAL_SLOTS - occupiedSlots);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two HC-SR04 Sensors**, **two Servos**, **I2C LCD**, and **HC-05** onto the canvas.
2. Connect HC-SR04 #1 to **GP14/GP15**, HC-SR04 #2 to **GP16/GP17**, Servos to **GP10/GP11**, LCD to **GP4/GP5**, and HC-05 to **GP0/GP1**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the entry sensor slider below 15 cm to open the gate, and check the logs over Bluetooth.

## Expected Output

Terminal (Bluetooth):
```
Occupied,Free
1,4
0,5
```

## Expected Canvas Behavior
* Startup: LCD reads `Filled: 0/5  Free: 5`. Gates are at 0°.
* Car arrives at entry sensor (dist1 = 10 cm): LCD reads `Welcome!`, Entry Servo rotates to 90° for 3 seconds, then returns to 0°. LCD updates to `Filled: 1/5  Free: 4`. Bluetooth prints `1,4`.
* Car arrives at exit sensor (dist2 = 10 cm): LCD reads `Thank You!`, Exit Servo rotates to 90° for 3 seconds, then returns to 0°. LCD updates to `Filled: 0/5  Free: 5`. Bluetooth prints `0,5`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.println(TOTAL_SLOTS - occupiedSlots)` | Transmits the remaining free slots over Bluetooth for parking garage occupancy trackers. |

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
