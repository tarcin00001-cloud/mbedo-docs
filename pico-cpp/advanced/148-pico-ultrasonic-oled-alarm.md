# 148 - Pico Ultrasonic OLED Alarm

Build an digital rangefinder collision alarm that displays proximity alerts on an OLED, sounds a warning buzzer, and switches a safety brake relay.

## Goal
Learn how to parse ultrasonic distance sensor data, format warning text frames on SSD1306 OLED displays, and trigger warning buzzers and safety relays.

## What You Will Build
A backup collision warning console:
- **HC-SR04 Sensor (Trig GP14, Echo GP15)**: Measures target distance.
- **SSD1306 OLED (GP4, GP5)**: Displays the measured distance, a visual proximity bar, and status alert messages.
- **Active Buzzer (GP10)**: Sounds a rapid warning beep if the distance drops below 30 cm.
- **Relay Module (GP11)**: Actuates a safety brake or shuts down a motor if the distance drops below 20 cm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Resistors | `resistor` | Optional | Yes (voltage divider) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 | Trig / Echo | GP14 / GP15 | Sensor lines |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| Active Buzzer | VCC (+) | GP10 | Warning beeper |
| Relay Module | IN | GP11 | Emergency brake line |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int TRIG_PIN   = 14;
const int ECHO_PIN   = 15;
const int BUZZER_PIN = 10;
const int RELAY_PIN  = 11;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

const int WARN_DISTANCE  = 30; // Proximity beep limit (cm)
const int BRAKE_DISTANCE = 20; // Emergency brake limit (cm)

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);

  digitalWrite(TRIG_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(RELAY_PIN, LOW); // Brake released

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
}

void loop() {
  // Generate trigger pulse
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  int distance = duration * 0.0343 / 2;

  // Limit range readings
  if (distance < 0 || distance > 100) {
    distance = 100;
  }

  // Invert scale: bar fills up as target gets closer
  int proximityWidth = 128 - (distance * 128 / 100);
  if (proximityWidth < 0) { proximityWidth = 0; }

  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(12, 3);
  display.print("COLLISION RADAR");

  // Display distance
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(10, 20);
  display.print("Range: ");
  display.print(distance);
  display.print(" cm");

  // Proximity alerts
  display.setCursor(10, 32);
  if (distance > 0 && distance < BRAKE_DISTANCE) {
    // Stage 2: Emergency brake
    digitalWrite(RELAY_PIN, HIGH); // Apply brake
    digitalWrite(BUZZER_PIN, HIGH); // Continuous alarm
    display.print("BRAKE: !!! ACTIVE !!!");
  } 
  else if (distance >= BRAKE_DISTANCE && distance < WARN_DISTANCE) {
    // Stage 1: Warning beep
    digitalWrite(RELAY_PIN, LOW); // Brake released
    display.print("BRAKE: WARNING BEEP");
    
    // Pulse warning beep
    digitalWrite(BUZZER_PIN, HIGH);
    delay(80);
    digitalWrite(BUZZER_PIN, LOW);
    delay(80);
  } 
  else {
    // Safe
    digitalWrite(RELAY_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);
    display.print("BRAKE: RELEASED/SAFE");
  }

  // Draw proximity bar graph border
  display.drawRect(0, 46, 128, 14, SSD1306_WHITE);
  if (proximityWidth > 0) {
    display.fillRect(2, 48, proximityWidth - 4, 10, SSD1306_WHITE);
  }
  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  delay(60); // Settle delay
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-SR04 Sensor**, **SSD1306 OLED**, **Active Buzzer**, and **Relay Module** onto the canvas.
2. Connect HC-SR04 to **GP14/GP15**, OLED to **GP4/GP5**, Buzzer to **GP10**, and Relay to **GP11**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the distance slider below 30 cm and 20 cm and observe the buzzer, brake relay, and OLED status change.

## Expected Output

Terminal:
```
Simulation active. Collision safety backup radar online.
```

## Expected Canvas Behavior
* Normal state (Distance > 30 cm): OLED reads `BRAKE: RELEASED/SAFE`. Relay and Buzzer are OFF.
* Warning zone (20–30 cm): OLED reads `BRAKE: WARNING BEEP`. Buzzer beeps rapidly, Relay remains OFF.
* Critical zone (< 20 cm): OLED reads `BRAKE: !!! ACTIVE !!!`. Relay turns ON (Brake applied), buzzer sounds continuously.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `distance < BRAKE_DISTANCE` | Evaluates if target is inside the critical brake boundary to apply emergency braking. |

## Hardware & Safety Concept: Multi-Stage Proximity Braking
Autonomous vehicles use multi-stage proximity alert systems. When approaching an obstacle:
1. **Warning Stage**: The system beeps and flashes warnings to alert the operator.
2. **Emergency Stage**: If the operator takes no action and the distance drops below the critical limit, the controller bypasses manual input and applies emergency braking directly using a relay to prevent a collision.

## Try This! (Challenges)
1. **Latching Lockout**: If the emergency brake is triggered, keep the relay active until a reset button (GP16) is pressed.
2. **Dynamic Beeping**: Make the buzzer beep faster as the distance drops.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Emergency brake triggers when path is clear | Sensor read error | If the Echo pin fails to report, the code reads 0. Ensure the safety condition checks `distance > 0` before applying the brake. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [76 - Pico Ultrasonic Alarm](../intermediate/76-pico-ultrasonic-alarm.md)
- [118 - Pico Ultrasonic OLED](../intermediate/118-pico-ultrasonic-oled.md)
- [133 - Pico Car Parking](133-pico-car-parking.md)
