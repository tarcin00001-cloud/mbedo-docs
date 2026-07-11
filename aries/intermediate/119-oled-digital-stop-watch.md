# 119 - OLED Digital Stop Watch

Build a high-precision digital stopwatch using start/stop control buttons and an SSD1306 OLED display using the VEGA ARIES v3 board.

## Goal
Learn how to implement button state-change detection (edge detection), track precise millisecond timing using `millis()`, and update a graphical display without loops.

## What You Will Build
A digital stopwatch display. Pressing the Start button on GPIO 16 starts the timer. The OLED display shows the elapsed time in seconds and tenths of a second. Pressing the Stop/Reset button on GPIO 17 pauses the timer; pressing it a second time when paused resets the timer back to 0.0 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| SSD1306 OLED Display | `oled` | Yes | Yes |
| Push Button (Start) | `button` | Yes | Yes |
| Push Button (Stop/Reset) | `button` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SSD1306 OLED | VCC | 3V3 | Red | OLED power supply (3.3V) |
| SSD1306 OLED | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA | SDA0 (GP17) | Blue | I2C Data line |
| SSD1306 OLED | SCL | SCL0 (GP16) | Yellow | I2C Clock line |
| Start Button | Pin 1 | GPIO 16 | Green | Input with internal pull-up |
| Start Button | Pin 2 | GND | Black | Ground reference |
| Stop/Reset Button | Pin 1 | GPIO 17 | White | Input with internal pull-up |
| Stop/Reset Button | Pin 2 | GND | Black | Ground reference |

> **Wiring tip:** Share the GND line using breadboard connections. Configure both push buttons with ARIES internal pull-up resistors (`INPUT_PULLUP`) so they connect directly to ground.

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

const int START_PIN = 16; // Start Button on GPIO 16
const int STOP_PIN = 17;  // Stop/Reset Button on GPIO 17

int isRunning = 0;
unsigned long startTime = 0;
unsigned long elapsedTime = 0;
unsigned long lastDisplayTime = 0;

int lastStartBtn = HIGH;
int lastStopBtn = HIGH;

void setup() {
  Wire.begin();
  display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR);
  display.clearDisplay();
  
  pinMode(START_PIN, INPUT_PULLUP);
  pinMode(STOP_PIN, INPUT_PULLUP);
  
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("Stopwatch Ready");
  display.display();
}

void loop() {
  int startBtn = digitalRead(START_PIN);
  int stopBtn = digitalRead(STOP_PIN);

  // Detect Start Button falling edge (Pressed)
  if (startBtn == LOW && lastStartBtn == HIGH) {
    if (isRunning == 0) {
      isRunning = 1;
      startTime = millis() - elapsedTime; // Offset by already elapsed time
    }
  }
  lastStartBtn = startBtn;

  // Detect Stop Button falling edge (Pressed)
  if (stopBtn == LOW && lastStopBtn == HIGH) {
    if (isRunning == 1) {
      isRunning = 0;
      elapsedTime = millis() - startTime; // Save elapsed time
    } else {
      elapsedTime = 0; // Reset timer when already paused
    }
  }
  lastStopBtn = stopBtn;

  // Accumulate elapsed milliseconds if running
  if (isRunning == 1) {
    elapsedTime = millis() - startTime;
  }

  // Smooth OLED refresh (every 100 ms to avoid I2C bus lag)
  if (millis() - lastDisplayTime >= 100) {
    display.clearDisplay();
    
    // Header
    display.setTextSize(1);
    display.setCursor(0, 0);
    display.print("Digital Stopwatch");

    // Format values
    unsigned long seconds = elapsedTime / 1000;
    unsigned long tenths = (elapsedTime % 1000) / 100;

    // Time Display
    display.setTextSize(2);
    display.setCursor(20, 20);
    display.print(seconds);
    display.print(".");
    display.print(tenths);
    display.print(" s");

    // Status Label
    display.setTextSize(1);
    display.setCursor(15, 48);
    if (isRunning == 1) {
      display.print("Status: RUNNING");
    } else if (elapsedTime > 0) {
      display.print("Status: PAUSED");
    } else {
      display.print("Status: READY");
    }

    display.display();
    lastDisplayTime = millis();
  }

  delay(20); // Fast state-machine loop scan
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **SSD1306 OLED**, and two **Push Buttons** onto the canvas.
2. Wire the OLED: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
3. Wire the Start Button: **Pin 1** to **GPIO 16** and **Pin 2** to **GND**.
4. Wire the Stop Button: **Pin 1** to **GPIO 17** and **Pin 2** to **GND**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.
8. Click the Start button to run the timer, click the Stop button to pause, and click Stop again to reset.

## Expected Output
Serial Monitor:
```
System Initialized.
OLED Stopwatch Ready.
Button Event: START (Running...)
Button Event: STOP (Paused at 4.2s)
Button Event: RESET (Ready)
```

## Expected Canvas Behavior
* Pressing the Start button starts counting seconds (e.g. `1.3 s`, `1.4 s`, ...) on the OLED display.
* Pressing the Stop button pauses the counter and changes the status text to `Status: PAUSED`.
* Pressing the Stop button again clears the counter to `0.0 s` and displays `Status: READY`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(START_PIN, INPUT_PULLUP)` | Pulls GP16 to 3.3V, allowing direct grounding on keypress. |
| `startBtn == LOW && lastStartBtn == HIGH` | Detects the transition from released (HIGH) to pressed (LOW). |
| `millis() - elapsedTime` | Calculates a virtual starting timestamp to account for resume offsets. |
| `elapsedTime / 1000` | Performs integer division to extract total whole seconds. |
| `(elapsedTime % 1000) / 100` | Extracts the first digit of the millisecond remainder (tenths of a second). |

## Hardware & Safety Concept
* **Timer Resolution Limits**: The `millis()` function returns the number of milliseconds elapsed since the board started up. While accurate enough for hobby stopwatches, the internal clock crystals on microcontrollers can experience drift over hours due to ambient thermal changes, meaning they are not suitable for certified athletic timing.
* **Display Bus Optimization**: Pushing a full screen buffer to an SSD1306 OLED over standard I2C at 100kHz takes about 20 ms. Sending screen updates constantly in the loop slows down button polling, causing missed presses. Refreshing the screen at a controlled 100 ms interval resolves this bottleneck.

## Try This! (Challenges)
1. **Lap Timer**: Connect a third button on GPIO 12. When pressed while running, freeze the current time on the screen for 2 seconds to show a "Lap Time" before resuming the active display.
2. **Beep Alarm**: Connect a buzzer on GPIO 14. Sound a short 100 ms beep whenever either button is pressed to confirm the action.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Pressing buttons has no effect | Button pins floating or wired incorrectly | Confirm that the buttons connect to ground on press and that `INPUT_PULLUP` is enabled in code. |
| Time jumps forward erratically when resuming | Start button bounce | Ensure the button state change logic (edge detection) is correctly comparing current and previous states. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [60 - OLED SSD1306 Display Setup (I2C)](60-oled-ssd1306-display-setup.md)
- [112 - DS3231 Real Time Clock Display](112-ds3231-real-time-clock-display.md)
