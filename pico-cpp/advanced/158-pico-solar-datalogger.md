# 158 - Pico Solar Datalogger

Build an automated solar tracker station that rotates a servo panel, measures panel output current via an ACS712 sensor, and streams power CSV logs to Serial.

## Goal
Learn how to read multiple analog inputs (LDR differential tracking, ACS712 current), drive servo motors, and format output streams for CSV telemetry data logging.

## What You Will Build
An efficiency-monitoring solar tracking system:
- **Left/Right LDRs (GP26, GP27)**: Detects the brightest light direction.
- **Servo Motor (GP10)**: Steers the solar panel platform.
- **ACS712 Current Sensor (GP28)**: Measures simulated panel output current.
- **16x2 I2C LCD (GP4, GP5)**: Displays the active panel angle and current load generation.
- **Serial Monitor**: Streams CSV-formatted log lines (e.g. `Angle,Current_mA`) every 3 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR Photoresistors | `ldr` | Yes (two LDRs) | Yes |
| Servo Motor | `servo` | Yes | Yes |
| ACS712 Current Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes (5A ACS712 module) |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Left LDR | Signal (Pin 2) | GP26 | Analog light tracking |
| Right LDR | Signal (Pin 2) | GP27 | Analog light tracking |
| Servo Motor | Signal | GP10 | Rotating platform control |
| ACS712 Sensor | OUT | GP28 | Analog current load sensor |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Servo.h>
#include <LiquidCrystal_I2C.h>

const int LDR_LEFT  = 26;
const int LDR_RIGHT = 27;
const int SERVO_PIN = 10;
const int ACS_PIN   = 28;

Servo trackerServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

int servoAngle = 90;
const int tolerance = 100;

void setup() {
  Serial.begin(9600);

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  trackerServo.attach(SERVO_PIN);
  trackerServo.write(servoAngle);
  
  pinMode(LDR_LEFT, INPUT);
  pinMode(LDR_RIGHT, INPUT);
  pinMode(ACS_PIN, INPUT);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Solar Logger");
  lcd.setCursor(0, 1);
  lcd.print("Online & Logging");

  // Print CSV Header to Serial
  Serial.println("Angle_deg,Current_mA");
  delay(1500);
}

void loop() {
  int valLeft  = analogRead(LDR_LEFT);
  int valRight = analogRead(LDR_RIGHT);
  int rawACS   = analogRead(ACS_PIN);

  // 1. Calculate LDR Differential and steer servo
  int diff = valLeft - valRight;
  int absDiff = diff;
  if (absDiff < 0) { absDiff = -absDiff; }

  if (absDiff > tolerance) {
    if (diff > 0) {
      servoAngle = servoAngle + 4;
      if (servoAngle > 180) { servoAngle = 180; }
    } else {
      servoAngle = servoAngle - 4;
      if (servoAngle < 0) { servoAngle = 0; }
    }
    trackerServo.write(servoAngle);
  }

  // 2. Calculate current from ACS712 (0-5A sensor)
  // Maps 0-4095 ADC to mA scale (0 to 5000 mA)
  float current_mA = rawACS * 5000.0 / 4095.0;

  // 3. Update LCD
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Angle: ");
  lcd.print(servoAngle);
  lcd.print(" deg");

  lcd.setCursor(0, 1);
  lcd.print("Load : ");
  lcd.print(current_mA, 0);
  lcd.print(" mA");

  // 4. Stream CSV log line
  Serial.print(servoAngle);
  Serial.print(",");
  Serial.println(current_mA, 2);

  delay(1000); // Check and log once per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two LDRs**, **Servo Motor**, **potentiometer** (for ACS712), and **I2C LCD** onto the canvas.
2. Connect Left LDR to **GP26**, Right LDR to **GP27**, Servo to **GP10**, potentiometer to **GP28**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the LDR values to steer the panel, and slide the current potentiometer to watch CSV logs print.

## Expected Output

Terminal:
```
Angle_deg,Current_mA
90,250.00
94,350.00
98,420.00
```

## Expected Canvas Behavior
* Startup: LCD reads `Angle: 90 deg` / `Load: 0 mA`.
* Slide LDR 1 up: Servo rotates left (angle rises), LCD updates.
* Slide current potentiometer up: LCD current load increases, CSV log logs value.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial.println("Angle_deg,Current_mA")` | Prints the CSV header to let external graphing programs parse data columns correctly. |

## Hardware & Safety Concept: Solar Power Monitoring
ACS712 hall-effect current sensors measure electrical current by detecting the magnetic field generated as current passes through an internal copper path. This provides galvanic isolation, protecting the 3.3V Raspberry Pi Pico from high-voltage spikes on the solar panel side.

## Try This! (Challenges)
1. **Critical Overload Cutoff**: Connect a relay on GP11 and shut off the charging line if the current exceeds 4000 mA.
2. **Dynamic Log rate**: Stream data faster (every 200 ms) when the panel is rotating to capture movements.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Current reads extreme numbers when quiet | Sensor offset drift | Hall-effect sensors are sensitive to stray magnetic fields. Add a tare/offset correction factor in code to zero the reading at startup. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [34 - Pico LDR Sensor](../../beginner/34-pico-ldr-sensor.md)
- [46 - Pico Pot Servo](../../beginner/46-pico-pot-servo.md)
- [134 - Pico Solar Tracker](134-pico-solar-tracker.md)
