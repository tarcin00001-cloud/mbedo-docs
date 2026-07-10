# 149 - Pico BMP180 OLED Alarm

Build an digital altimeter alarm station that sounds alerts and switches a safety relay if relative altitude limits are breached.

## Goal
Learn how to parse barometric altitude changes from I2C sensors (BMP180), display coordinates and limit states on OLED screens, and actuate safety switches.

## What You Will Build
An aviation altitude warning console:
- **BMP180 Sensor (GP4, GP5)**: Measures altitude based on air pressure.
- **SSD1306 OLED (GP4, GP5)**: Displays temperature, pressure, current altitude, and warning status.
- **Active Buzzer (GP14)**: Sounds an alert if the altitude exceeds the safety setpoint (e.g. 150 meters).
- **Relay Module (GP10)**: Activates a safety valve or backup system during alarms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| BMP180 Sensor | SDA / SCL | GP4 / GP5 | Shared I2C sensor bus |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| Active Buzzer | VCC (+) | GP14 | Alarm buzzer |
| Relay Module | IN | GP10 | Safety actuator line |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_BMP085.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

Adafruit_BMP085 bmp;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

const int BUZZER_PIN = 14;
const int RELAY_PIN  = 10;

// High altitude safety limit (meters)
const float ALTITUDE_LIMIT = 150.0;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);

  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(RELAY_PIN, LOW);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  if (!bmp.begin()) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(10, 20);
    display.print("BMP180 Error!");
    display.display();
    while (1);
  }
}

void loop() {
  float temp = bmp.readTemperature();
  long pressure = bmp.readPressure();
  float altitude = bmp.readAltitude();

  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(18, 3);
  display.print("ALTITUDE WARNING");

  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);

  // Row 1: Altitude
  display.setCursor(10, 20);
  display.print("Alt : ");
  display.print(altitude, 1);
  display.print(" m");

  // Row 2: Pressure
  display.setCursor(10, 32);
  display.print("Pres: ");
  display.print(pressure / 100);
  display.print(" hPa");

  // Row 3: Safety Alarm Status
  display.setCursor(10, 48);
  if (altitude > ALTITUDE_LIMIT) {
    // ALTITUDE BREACH
    digitalWrite(BUZZER_PIN, HIGH);
    digitalWrite(RELAY_PIN, HIGH);
    display.print("STATUS: !!! BREACH !!!");
  } else {
    // SECURE
    digitalWrite(BUZZER_PIN, LOW);
    digitalWrite(RELAY_PIN, LOW);
    display.print("STATUS: SECURE");
  }

  // Outer border frame
  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  delay(1000); // Update once per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **BMP180 Sensor**, **SSD1306 OLED**, **Active Buzzer**, and **Relay Module** onto the canvas.
2. Connect BMP180 and OLED to **GP4/GP5** in parallel. Connect Buzzer to **GP14**, and Relay to **GP10**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the sensor pressure slider to simulate high altitude (low pressure values) and watch the alarm trigger.

## Expected Output

Terminal:
```
Simulation active. Altimeter limit warning station online.
```

## Expected Canvas Behavior
* Normal state (Altitude < 150m): OLED reads `STATUS: SECURE`. Relay and Buzzer are OFF.
* Altitude breach (> 150m): OLED reads `STATUS: !!! BREACH !!!`. Relay turns ON, buzzer beeps.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `altitude > ALTITUDE_LIMIT` | Checks if the calculated altitude exceeds the safety threshold to trigger safety shutdown relays. |

## Hardware & Safety Concept: Aviation Altimeter Alerts
In drone and aircraft avionics, altitude warning indicators verify that the aircraft remains within safe flight ceilings. If a drone drifts above legal or mechanical altitude limits, safety algorithms activate warning chimes and trigger throttle reduction overrides (safety relays) to force the aircraft to descend automatically.

## Try This! (Challenges)
1. **Low Altitude Alert**: Sound the alarm if the altitude drops below 10 meters (simulating terrain collision risk).
2. **Pressure Units**: Add code to toggle displaying pressure in inches of Mercury (inHg) on button click.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Altitude drifts constantly | Local weather changes | Barometric altimeters calculate altitude based on absolute pressure. Adjust target thresholds to account for daily weather changes. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [78 - Pico BMP180 Serial](../intermediate/78-pico-bmp180-serial.md)
- [79 - Pico BMP180 LCD](../intermediate/79-pico-bmp180-lcd.md)
- [119 - Pico BMP180 OLED](../intermediate/119-pico-bmp180-oled.md)
