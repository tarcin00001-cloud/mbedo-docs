# 107 - ESP32 Fire Warning Station

Build a safety fire alert node that monitors for open flames using an IR flame sensor, displays status messages on an SSD1306 OLED screen, and sounds an emergency siren when a fire is detected.

## Goal
Learn how to build multi-device safety stations incorporating high-priority graphics displays, fast sensor queries, and emergency buzzer sirens.

## What You Will Build
A digital flame sensor is connected to GPIO 4. A buzzer is connected to GPIO 15, and an SSD1306 OLED screen on I2C (GPIO 21/22). Under normal conditions, the OLED displays "SYSTEM SAFE". When a flame is detected, the OLED flashes a warning "!! FIRE ALARM !!" and the buzzer sounds a rapid alarm siren.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Flame Sensor Module (IR Type) | `flame_sensor` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Flame Sensor | DO | GPIO4 | Yellow | Flame digital input |
| Flame Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| SSD1306 OLED | VCC / GND | 3V3 / GND | Red / Black | OLED power rails |
| SSD1306 OLED | SDA | GPIO21 | Blue | I2C Data line |
| SSD1306 OLED | SCL | GPIO22 | Yellow | I2C Clock line |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Ground reference |

> **Wiring tip:** Standard flame sensors output active-LOW digital logic (LOW = fire detected). Connect DO to GPIO 4.

## Code
```cpp
// Fire Warning Station
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

const int FLAME_PIN = 4;
const int BUZZER_PIN = 15;

void setup() {
  Serial.begin(115200);
  
  pinMode(FLAME_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED Init failed!");
    while (1) {}
  }
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("Fire Monitor Active");
  display.display();
  
  delay(1500);
}

void loop() {
  // Read digital output of flame sensor (LOW = flame detected)
  bool fireDetected = (digitalRead(FLAME_PIN) == LOW);
  
  if (fireDetected) {
    Serial.println("!! CRITICAL: FIRE DETECTED !!");
    
    // Trigger audio siren and flashing OLED screen
    // Flashing Loop
    for (int i = 0; i < 2; i++) {
      // Screen version A (Inverted: Black on White)
      display.clearDisplay();
      display.fillScreen(SSD1306_WHITE);
      display.setTextColor(SSD1306_BLACK);
      display.setTextSize(2);
      display.setCursor(15, 10);
      display.print("!! FIRE !!");
      display.setTextSize(1);
      display.setCursor(20, 45);
      display.print("EVACUATE AREA");
      display.display();
      
      digitalWrite(BUZZER_PIN, HIGH);
      delay(200);
      
      // Screen version B (Normal: White on Black)
      display.clearDisplay();
      display.setTextColor(SSD1306_WHITE);
      display.setTextSize(2);
      display.setCursor(15, 10);
      display.print("!! FIRE !!");
      display.setTextSize(1);
      display.setCursor(20, 45);
      display.print("EVACUATE AREA");
      display.display();
      
      digitalWrite(BUZZER_PIN, LOW);
      delay(200);
    }
  } else {
    // Normal monitoring state
    digitalWrite(BUZZER_PIN, LOW);
    
    display.clearDisplay();
    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);
    
    // Draw status HUD frame
    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.setCursor(10, 15);
    display.print("Status: SAFE");
    display.setCursor(10, 35);
    display.print("No fire detected");
    display.display();
    
    delay(500); // 2Hz poll when idle
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Flame Sensor**, **Buzzer**, and **SSD1306 OLED** onto the canvas.
2. Wire Flame DO to **GPIO4**, Buzzer to **GPIO15**, and OLED to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Click the flame sensor widget to simulate a fire. Watch the OLED flash and the buzzer sound.

## Expected Output
Serial Monitor:
```
!! CRITICAL: FIRE DETECTED !!
!! CRITICAL: FIRE DETECTED !!
```

OLED Display Layout (during normal state):
```
┌────────────────┐
│  Status: SAFE  │
│  No fire det.  │
└────────────────┘
```

OLED Display Layout (during fire alarm):
```
[ !! FIRE !! ] (Flashes white/black screen backgrounds rapidly)
EVACUATE AREA
```

## Expected Canvas Behavior
* During normal monitoring, the OLED widget displays a boundary frame box with "Status: SAFE".
* Triggering the virtual flame sensor widget immediately sounds the buzzer siren and causes the OLED widget to flash inverted text back and forth.
* The alarm shuts off when the flame widget is deactivated.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `display.fillScreen(SSD1306_WHITE)` | Fills the entire screen buffer with white pixels for an inverted warning screen. |
| `display.setTextColor(SSD1306_BLACK)` | Switches text color to black to draw over the white background. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives the active buzzer siren. |

## Hardware & Safety Concept: Fire Monitoring Redundancy
Fire warning devices utilize optical infrared sensors because flames generate infrared wavelengths. Because sunlight or incandescent bulbs also emit IR, industrial safety monitors require triple-sensor voting schemes (e.g. UV sensor, IR sensor, and thermal sensor must all trigger together) to confirm a fire before dumping extinguishing foam.

## Try This! (Challenges)
1. **Relay valve shutdown**: Add a relay on GPIO 13 that shuts down a gas valve if the alarm is triggered.
2. **Dynamic Alarm Pitch**: If using a passive buzzer, create a rising-pitch siren (similar to emergency vehicles).
3. **Scan Log database**: Display the total duration of the fire alarm event on the screen.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Light bulb interference | Keep sensor away from incandescent bulbs; adjust the trimmer screw |
| OLED displays garbled noise | Missing `display.display()` | Ensure `display.display()` is called at the end of each screen update cycle |
| Buzzer does not sound | Incorrect output pin mapping | Verify active buzzer is connected to GPIO 15 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [44 - ESP32 Flame Sensor Digital Fire Alert](../beginner/44-esp32-flame-sensor-digital-fire-alert.md)
- [60 - ESP32 OLED SSD1306 Display Setup](60-esp32-oled-ssd1306-display-setup.md)
- [98 - ESP32 Gas Leakage Alarm Node](98-esp32-gas-leakage-alarm-node.md)
