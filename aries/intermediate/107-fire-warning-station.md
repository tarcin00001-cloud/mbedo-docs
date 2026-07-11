# 107 - Fire Warning Station

Build an emergency fire warning station that uses a digital flame sensor to detect fire, triggering a high-decibel buzzer and updating an OLED display with warning statuses.

## Goal
Learn how to interface infrared flame sensors, control active sound indicators, and render large text warnings on an I2C OLED display in real-time.

## What You Will Build
An automatic fire alarm. When the flame sensor detects infrared radiation from fire, the buzzer starts sounding continuously, and the OLED display flashes a large `!!!FIRE!!!` warning. When no flame is present, the buzzer shuts off and the OLED displays `Status: SECURE`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Flame Sensor | `flame_sensor` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| SSD1306 OLED Display | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Flame Sensor | VCC | 3V3 | Red | Sensor operating voltage (3.3V) |
| Flame Sensor | GND | GND | Black | Ground reference |
| Flame Sensor | D0 (Digital Out)| GPIO 17 | Green | Digital flame signal (Active LOW) |
| Active Buzzer | VCC | GPIO 14 | Orange | Output high triggers alarm |
| Active Buzzer | GND | GND | Black | Ground reference |
| SSD1306 OLED | VCC | 3V3 | Red | OLED power supply (3.3V) |
| SSD1306 OLED | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA | SDA0 (GP17) | Blue | I2C Data line |
| SSD1306 OLED | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** Standard digital flame sensor modules output a logic `LOW` when a fire source is detected, and `HIGH` when the environment is safe. Verify your module specifications.

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

const int FLAME_PIN = 17;   // Flame sensor on GPIO 17
const int BUZZER_PIN = 14;  // Active buzzer on GPIO 14

int lastFireState = -1;

void setup() {
  Wire.begin();
  display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR);
  display.clearDisplay();
  
  pinMode(FLAME_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW); // Start with buzzer OFF
  
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("Fire Warning Station");
  display.display();
}

void loop() {
  // Read state: Active LOW (0 = Flame detected, 1 = Safe)
  int fireState = digitalRead(FLAME_PIN);

  // Update only on state change to prevent OLED wear and lag
  if (fireState != lastFireState) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setCursor(0, 0);
    display.print("Fire Warning Station");

    if (fireState == LOW) {
      // Flame detected: sound alarm and show alert
      digitalWrite(BUZZER_PIN, HIGH);
      
      display.setTextSize(2);
      display.setCursor(10, 30);
      display.print("!!!FIRE!!!");
    } else {
      // Safe: turn off alarm and show secure status
      digitalWrite(BUZZER_PIN, LOW);
      
      display.setTextSize(1);
      display.setCursor(10, 35);
      display.print("Status: SECURE");
    }
    display.display();
    lastFireState = fireState;
  }
  delay(100); // Polling delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Flame Sensor**, **Active Buzzer**, and **SSD1306 OLED** components onto the canvas.
2. Wire the Flame Sensor: **VCC** to **3V3**, **GND** to **GND**, and **D0** to **GPIO 17**.
3. Wire the Active Buzzer: **VCC** to **GPIO 14** and **GND** to **GND**.
4. Wire the OLED: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Flame Sensor: SAFE
...
Flame Sensor: FIRE DETECTED - ALARM ACTIVE!
```

## Expected Canvas Behavior
* Triggering the simulated flame sensor (representing fire exposure) immediately sounds a continuous high-pitched alarm on the buzzer.
* The OLED display clears its status and shows a bold flashing `!!!FIRE!!!` warning.
* Disabling the flame sensor stops the buzzer and displays `Status: SECURE` on the OLED screen.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(FLAME_PIN, INPUT)` | Configures pin GPIO 17 as input to read digital outputs from the flame sensor. |
| `digitalRead(FLAME_PIN)` | Reads the logic state (0 for fire, 1 for safe). |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives GPIO 14 high to activate the buzzer alarm. |
| `display.print("!!!FIRE!!!")` | Draws the warning message in the local buffer. |
| `display.display()` | Pushes changes to the OLED controller over I2C. |

## Hardware & Safety Concept
* **Infrared Flame Detection**: Flame sensors use a photodiode sensitive to infrared wavelengths (typically 760nm to 1100nm) emitted by fire combustion. Because sunlight and incandescent bulbs also emit infrared light, position the sensor away from these sources to avoid false triggering.
* **Fail-Safe Warning**: Fire detection systems must remain functional under critical conditions. Always power the alarm circuitry from a stable external battery bank or backup power supply.

## Try This! (Challenges)
1. **Flashing OLED Text**: When fire is detected, toggle the text background state between normal and inverted using `display.invertDisplay(true)` and `display.invertDisplay(false)` in the main loop to create a flashing screen effect.
2. **Onboard LED Warning**: Flash the onboard Red LED (`LED_R` on GPIO 23) rapidly at 50 ms intervals while the alarm is active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The alarm sounds constantly when no fire is present | Sensor sensitivity set too high | Adjust the small trimmer potentiometer on the flame sensor board counter-clockwise to raise the detection threshold. |
| OLED shows garbled text or does not update | Incorrect I2C bus wiring | Confirm that SDA and SCL connect to ARIES pins GP17 and GP16 respectively. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [60 - OLED SSD1306 Display Setup (I2C)](60-oled-ssd1306-display-setup.md)
- [98 - Gas Leakage Alarm Node](98-gas-leakage-alarm-node.md)
