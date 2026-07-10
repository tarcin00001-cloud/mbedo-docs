# 101 - ESP32 Automatic Barrier Gate

Build an automated security gate system that detects approaching vehicles using an IR obstacle sensor, sweeps a servo motor to open the barrier, and displays welcome messages on a 16x2 I2C LCD.

## Goal
Learn how to coordinate proximity inputs, servo gate positioners, and character LCD status readouts to build automated entry-point access systems.

## What You Will Build
An IR obstacle sensor is connected to GPIO 4. A servo motor representing the barrier gate arm is connected to GPIO 13. A 16x2 I2C LCD displays status messages. When a vehicle is detected, the LCD displays "Access Granted / Welcome!", and the servo rotates to 90° (open). After 4 seconds, the gate closes (0°), and the LCD returns to "Ready" status.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| IR Obstacle Sensor Module | `ir_sensor` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Sensor | DO | GPIO4 | Yellow | Vehicle proximity sensor |
| IR Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| Servo Motor | Signal | GPIO13 | Orange | Gate arm control |
| Servo Motor | VCC / GND | 5V / GND | Red / Black | Power rails |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock |

> **Wiring tip:** Standard IR obstacle sensors output a digital LOW (active-LOW) signal when an object is detected. Connect the DO pin directly to GPIO 4.

## Code
```cpp
// Automatic Barrier Gate
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <ESP32Servo.h>

const int IR_PIN = 4;
const int SERVO_PIN = 13;

LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo gateServo;

bool gateOpen = false;

void setup() {
  Serial.begin(115200);
  
  pinMode(IR_PIN, INPUT); // IR sensor module drives output actively
  
  gateServo.attach(SERVO_PIN);
  gateServo.write(0); // Start with gate CLOSED (0 degrees)
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Barrier Gate");
  lcd.setCursor(0, 1);
  lcd.print("System Active");
  
  delay(1500);
  lcd.clear();
}

void loop() {
  // IR sensor is active-LOW (LOW = obstacle/vehicle detected)
  bool vehicleDetected = (digitalRead(IR_PIN) == LOW);
  
  if (vehicleDetected && !gateOpen) {
    Serial.println("Vehicle detected! Opening gate...");
    
    // Update LCD
    lcd.setCursor(0, 0);
    lcd.print("Access Granted  ");
    lcd.setCursor(0, 1);
    lcd.print("Welcome!        ");
    
    // Open Gate
    gateServo.write(90);
    gateOpen = true;
    
    // Keep gate open for 4 seconds
    delay(4000);
    
    Serial.println("Closing gate...");
    lcd.setCursor(0, 0);
    lcd.print("Gate Closing    ");
    lcd.setCursor(0, 1);
    lcd.print("Please wait...  ");
    
    // Close Gate
    gateServo.write(0);
    delay(1000); // Allow transit time for gate arm
    gateOpen = false;
    lcd.clear();
  } 
  
  if (!gateOpen) {
    // Normal ready display
    lcd.setCursor(0, 0);
    lcd.print("Gate: CLOSED    ");
    lcd.setCursor(0, 1);
    lcd.print("Approach Sensor ");
  }
  
  delay(100); // Poll sensor at 10Hz
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **IR Sensor**, **Servo**, and **16x2 I2C LCD** onto the canvas.
2. Wire IR Sensor DO to **GPIO4**, Servo signal to **GPIO13**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Click the IR sensor widget on the canvas to trigger a vehicle scan. Watch the gate open and close in sequence.

## Expected Output
Serial Monitor:
```
Vehicle detected! Opening gate...
Closing gate...
```

LCD Display (during open phase):
```
Access Granted
Welcome!
```

## Expected Canvas Behavior
* While the IR sensor is clear, the servo stays at 0° and the LCD displays "Gate: CLOSED".
* Clicking the IR sensor widget moves the servo arm to 90°, and the LCD displays the welcome message.
* After 4 seconds, the servo arm sweeps back to 0°, and the screen returns to "Gate: CLOSED".

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `digitalRead(IR_PIN) == LOW` | Evaluates vehicle proximity (active-LOW logic). |
| `gateServo.write(90)` | Rotates the gate barrier arm to the open position. |
| `delay(4000)` | Keeps the gate open to allow the vehicle time to drive through. |

## Hardware & Safety Concept: Gate Safety Interlocks
Automated parking gates must include safety loops (inductive loop detectors under the asphalt or photo-beam sensors across the driveway). If a car stops directly under the barrier arm, these safety sensors detect it and override the timer, keeping the gate open to prevent the arm from lowering onto the vehicle.

## Try This! (Challenges)
1. **Photo-beam Interlock Safety**: Add a second IR sensor (GPIO 15) acting as a safety beam. Prevent the gate from closing if this beam is blocked.
2. **Transit Counter Display**: Keep track of the total number of vehicles that have entered and display the counter on the LCD.
3. **Emergency Override Button**: Add a button on GPIO 12 that keeps the gate open indefinitely until pressed again.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Gate opens instantly and stays open | IR sensor sensitivity too high | Adjust the onboard sensor trim potentiometer to reduce range |
| Servo wiggles but doesn't reach 90° | Low power current | Connect servo VCC to the 5V (Vin) pin |
| LCD does not clear old letters | Buffer spaces missing in prints | Append blank spaces to fill all 16 character columns |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [37 - ESP32 IR Obstacle Sensor Print](../beginner/37-esp32-ir-obstacle-sensor-print.md)
- [51 - ESP32 Servo Motor 180° Sweep](51-esp32-servo-motor-180-sweep.md)
- [87 - ESP32 MFRC522 RFID Servo Door Unlock](87-esp32-mfrc522-rfid-servo-door-unlock.md)
