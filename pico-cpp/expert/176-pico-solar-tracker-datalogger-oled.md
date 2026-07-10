# 176 - Pico Solar Tracker Datalogger OLED

Build an advanced solar tracking terminal that steers a servo platform, measures current load, displays diagnostics on an OLED, and logs data over Bluetooth.

## Goal
Learn how to monitor multiple analog inputs (LDR differential tracking, ACS712 current), drive servo motors, design graphical OLED dashboards, and transmit Bluetooth CSV data logs.

## What You Will Build
A wireless efficiency-monitoring solar tracking node:
- **Left/Right LDRs (GP26, GP27)**: Tracks the brightest light direction.
- **Servo Motor (GP10)**: Steers the solar panel platform.
- **ACS712 Current Sensor (GP28)**: Measures simulated panel output current.
- **SSD1306 OLED (GP4, GP5)**: Displays active panel angle, current load, and power output.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits CSV logs wirelessly (e.g. `Angle,Current_mA`).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR Photoresistors | `ldr` | Yes (two LDRs) | Yes |
| Servo Motor | `servo` | Yes | Yes |
| ACS712 Current Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes (5A ACS712 module) |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Left LDR | Signal (Pin 2) | GP26 | Analog light tracking |
| Right LDR | Signal (Pin 2) | GP27 | Analog light tracking |
| Servo Motor | Signal | GP10 | Rotating platform control |
| ACS712 Sensor | OUT | GP28 | Analog current load sensor |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Servo.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int LDR_LEFT  = 26;
const int LDR_RIGHT = 27;
const int SERVO_PIN = 10;
const int ACS_PIN   = 28;

Servo trackerServo;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

int servoAngle = 90;
const int tolerance = 100;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  trackerServo.attach(SERVO_PIN);
  trackerServo.write(servoAngle);
  
  pinMode(LDR_LEFT, INPUT);
  pinMode(LDR_RIGHT, INPUT);
  pinMode(ACS_PIN, INPUT);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  // Print CSV Header to Bluetooth
  Serial1.println("Angle_deg,Current_mA");
  delay(1000);
}

void loop() {
  int valLeft  = analogRead(LDR_LEFT);
  int valRight = analogRead(LDR_RIGHT);
  int rawACS   = analogRead(ACS_PIN);

  // 1. Calculate LDR Differential and steer servo
  int diff = valLeft - valRight;
  int absDiff = diff;
  if (absDiff < 0) { absDiff = -absDiff; }

  char steerState[10] = "LOCKED";

  if (absDiff > tolerance) {
    if (diff > 0) {
      servoAngle = servoAngle + 4;
      if (servoAngle > 180) { servoAngle = 180; }
      steerState[0] = 'L'; steerState[1] = 'E'; steerState[2] = 'F'; steerState[3] = 'T'; steerState[4] = '\0';
    } else {
      servoAngle = servoAngle - 4;
      if (servoAngle < 0) { servoAngle = 0; }
      steerState[0] = 'R'; steerState[1] = 'I'; steerState[2] = 'G'; steerState[3] = 'H'; steerState[4] = 'T'; steerState[5] = '\0';
    }
    trackerServo.write(servoAngle);
  }

  // 2. Convert current: maps 12-bit ADC (0-4095) to 0-5000mA
  float current_mA = rawACS * 5000.0 / 4095.0;

  // 3. Update OLED Screen
  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(20, 3);
  display.print("SOLAR TELEMETRY");

  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  
  // Row 1: Angle & steer status
  display.setCursor(8, 20);
  display.print("Angle: ");
  display.print(servoAngle);
  display.print(" deg (");
  display.print(steerState);
  display.print(")");

  // Row 2: Load current
  display.setCursor(8, 34);
  display.print("Load : ");
  display.print(current_mA, 0);
  display.print(" mA");

  // Row 3: Power output in milliwatts (simulated at 5V panel)
  display.setCursor(8, 48);
  display.print("Power: ");
  display.print(current_mA * 5.0, 1);
  display.print(" mW");

  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  // 4. Stream CSV logs over Bluetooth
  Serial1.print(servoAngle);
  Serial1.print(",");
  Serial1.println(current_mA, 2);

  delay(2000); // Check and log once every 2 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two LDRs**, **Servo Motor**, **potentiometer** (for ACS712), **SSD1306 OLED**, and **HC-05** onto the canvas.
2. Connect Left LDR to **GP26**, Right LDR to **GP27**, Servo to **GP10**, potentiometer to **GP28**, OLED to **GP4/GP5**, and HC-05 to **GP0/GP1**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust LDR sensors to steer the panel, and slide current potentiometer to watch CSV logs print.

## Expected Output

Terminal (Bluetooth):
```
Angle_deg,Current_mA
90,250.00
94,350.00
98,420.00
```

## Expected Canvas Behavior
* Startup: OLED reads `Angle: 90 deg (LOCKED)` / `Load: 0 mA` / `Power: 0.0 mW`.
* Slide Left LDR up: Servo rotates left (angle rises), OLED updates.
* Slide current potentiometer up: OLED current load and power updates, CSV log logs values.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.println(current_mA, 2)` | Transmits current load coordinates wirelessly over Bluetooth to the monitoring terminal. |

## Hardware & Safety Concept: Solar Power Monitoring
ACS712 hall-effect current sensors measure electrical current by detecting the magnetic field generated as current passes through an internal copper path. This provides galvanic isolation, protecting the 3.3V Raspberry Pi Pico from high-voltage spikes on the solar panel side.

## Try This! (Challenges)
1. **Critical Overload Cutoff**: Connect a relay on GP11 and shut off the charging line if the current exceeds 4000 mA.
2. **Dynamic Log rate**: Stream data faster when the panel is actively steering.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Current reads static high value | Potentiometer wiper noise | Ensure analog sensor references are wired correctly to GP28 and scaled properly in the code. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [134 - Pico Solar Tracker](../advanced/134-pico-solar-tracker.md)
- [153 - Pico Solar Tracker OLED](../advanced/153-pico-solar-tracker-oled.md)
- [158 - Pico Solar Datalogger](../advanced/158-pico-solar-datalogger.md)
