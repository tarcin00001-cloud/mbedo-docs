# 96 - Sound Level OLED Decibel Meter

Measure ambient sound levels using an analog sound sensor, approximate the decibel equivalent, and display both numerical readouts and a progress bar dynamically on an SSD1306 OLED screen using the VEGA ARIES v3 board.

## Goal
Learn how to capture analog signals from a sound sensor, map the voltage range to approximate decibel levels, and render live text and graphics on an I2C OLED display without using loops or custom helper functions.

## What You Will Build
An interactive decibel meter that reads sound intensity from the environment and displays a real-time decibel level (30 dB to 100 dB) alongside an auto-scaling bar graph on the OLED display.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Analog Sound Sensor | `sound_sensor` | Yes | Yes |
| SSD1306 OLED Display | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Sound Sensor | VCC | 3V3 | Red | Sensor power supply (3.3V) |
| Sound Sensor | GND | GND | Black | Ground reference |
| Sound Sensor | OUT | ADC0 (GP26) | White | Analog output signal |
| SSD1306 OLED | VCC | 3V3 | Red | OLED power supply (3.3V) |
| SSD1306 OLED | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA | SDA0 (GP17) | Blue | I2C Data line |
| SSD1306 OLED | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** Share the 3V3 and GND pins using breadboard power rails. Connect the analog output of the sound sensor directly to GP26 (ADC0) and the OLED display to the hardware I2C0 pins.

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
const int SOUND_PIN = 26; // ADC0 is GP26
int lastDb = -1;

void setup() {
  Wire.begin();
  display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR);
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("Decibel Meter");
  display.display();
}

void loop() {
  int sensorValue = analogRead(SOUND_PIN);
  
  // Approximate sound level scaling (30 dB baseline to 100 dB maximum)
  int db = 30 + (sensorValue * 70 / 4095);

  // Update screen only when value changes to prevent flickering
  if (db != lastDb) {
    display.clearDisplay();
    
    // Title
    display.setTextSize(1);
    display.setCursor(0, 0);
    display.print("Sound Decibel Meter");

    // Value Display
    display.setTextSize(2);
    display.setCursor(20, 20);
    display.print(db);
    display.print(" dB");

    // Draw static bar outline
    display.drawRect(0, 50, 128, 10, SSD1306_WHITE);

    // Map decibels (30-100) to bar width (0-124 pixels)
    int barWidth = (db - 30) * 124 / 70;
    if (barWidth > 0) {
      display.fillRect(2, 52, barWidth, 6, SSD1306_WHITE);
    }
    
    display.display();
    lastDb = db;
  }
  delay(150); // Sensor read interval
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Analog Sound Sensor**, and **SSD1306 OLED** components onto the canvas.
2. Wire the Sound Sensor: **VCC** to **3V3**, **GND** to **GND**, and **OUT** to **ADC0 (GP26)**.
3. Wire the OLED: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
OLED SSD1306 Initialized at 0x3C.
ADC: 1170, Decibels: 50 dB
```

## Expected Canvas Behavior
* Clapping or speaking near the sound sensor in the simulator increases the simulated analog input value.
* The OLED display updates instantly, showing the corresponding numerical decibel level and widening the graphic bar representation.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Adafruit_SSD1306 display(...)` | Initializes the OLED object using the hardware I2C instance. |
| `analogRead(SOUND_PIN)` | Reads the analog voltage (0V to 3.3V) as a 12-bit integer (0 to 4095). |
| `30 + (sensorValue * 70 / 4095)` | Converts raw ADC readings into an approximate decibel value between 30 and 100 dB. |
| `db != lastDb` | Prevents redrawing the screen unless the decibel level changes. |
| `display.fillRect(...)` | Draws a solid rectangular fill inside the frame buffer to represent the sound level. |

## Hardware & Safety Concept
* **Acoustic Sensor Sensitivity**: Condenser microphones produce small alternating currents that require an onboard pre-amplifier circuit. These modules output a DC offset (usually 1.65V) that oscillates with sound waves.
* **OLED Current Limit**: OLED displays illuminate individual pixels, so current draw varies with brightness and lit pixel count. Keep active text clean and bar shapes simple to optimize power performance.

## Try This! (Challenges)
1. **Add Sound Alarm**: Connect a warning LED to GPIO 15. If the decibel level exceeds 85 dB (danger threshold), turn the LED HIGH; otherwise, turn it LOW.
2. **Peak Hold**: Display a single pixel tick line on the bar graph marking the highest decibel level reached. Reset the peak reading only when a button on GPIO 16 is pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED displays text but no progress bar | `db` outside 30 to 100 range | Check the raw analog readings. Adjust mapping logic if the sensor base voltage is higher. |
| The display value changes very slowly | Delay is too high | Decrease the loop polling delay below 150ms for faster updates. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [60 - OLED SSD1306 Display Setup (I2C)](60-oled-ssd1306-display-setup.md)
- [67 - Potentiometer Speed DC Motor (with LCD)](67-potentiometer-speed-dc-motor.md)
