# 119 - ESP32 OLED Digital Stop Watch

Build a high-precision digital stopwatch that displays elapsed minutes, seconds, and centiseconds on an SSD1306 OLED screen, controlled by Start/Pause and Reset buttons.

## Goal
Learn how to track precise elapsed time using non-blocking clock functions (`millis()`), build a state machine (RUNNING, PAUSED, RESET), and update display coordinates.

## What You Will Build
An SSD1306 OLED screen is connected via I2C. Two push buttons are connected: Button A (Start/Pause) on GPIO 4, and Button B (Reset) on GPIO 15. The screen displays the running stopwatch time in `MM:SS:CC` (Minutes:Seconds:Centiseconds) format.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |
| Tactile Push Buttons (2) | `button` | Yes | Yes |
| 10 kΩ Resistors (2) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SSD1306 OLED | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| SSD1306 OLED | SDA | GPIO21 | Blue | I2C Data line |
| SSD1306 OLED | SCL | GPIO22 | Yellow | I2C Clock line |
| Button A (Start) | Pin 1 / Pin 2 | 3V3 / GPIO4 | Red / Yellow | Start / Pause trigger |
| Resistor A (10k) | Leg 1 / Leg 2 | GPIO4 / GND | White / Black | Pull-down for Button A |
| Button B (Reset) | Pin 1 / Pin 2 | 3V3 / GPIO15 | Red / Green | Reset trigger |
| Resistor B (10k) | Leg 1 / Leg 2 | GPIO15 / GND | White / Black | Pull-down for Button B |

> **Wiring tip:** Connect both button pull-down resistors to GND to hold the signal pins LOW when the buttons are open.

## Code
```cpp
// OLED Digital Stop Watch
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

const int BTN_START = 4;
const int BTN_RESET = 15;

// Stopwatch States
enum State { RESET, RUNNING, PAUSED };
State currentState = RESET;

unsigned long startTime = 0;
unsigned long elapsedPausedTime = 0;
unsigned long totalElapsedTime = 0;

bool lastStartBtn = LOW;
bool lastResetBtn = LOW;

void setup() {
  Serial.begin(115200);
  
  pinMode(BTN_START, INPUT);
  pinMode(BTN_RESET, INPUT);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED allocation failed!");
    while (1) {}
  }
  
  display.clearDisplay();
  updateDisplay(0, 0, 0);
  Serial.println("Stopwatch Online.");
}

void loop() {
  bool startBtn = (digitalRead(BTN_START) == HIGH);
  bool resetBtn = (digitalRead(BTN_RESET) == HIGH);
  unsigned long now = millis();
  
  // 1. Handle Start/Pause Button Edge
  if (startBtn && !lastStartBtn) {
    delay(50); // Simple debounce
    if (currentState == RESET) {
      startTime = now;
      currentState = RUNNING;
      Serial.println("Stopwatch: RUNNING");
    } 
    else if (currentState == RUNNING) {
      elapsedPausedTime += (now - startTime);
      currentState = PAUSED;
      Serial.println("Stopwatch: PAUSED");
    } 
    else if (currentState == PAUSED) {
      startTime = now;
      currentState = RUNNING;
      Serial.println("Stopwatch: RESUMED");
    }
  }
  lastStartBtn = startBtn;
  
  // 2. Handle Reset Button Edge
  if (resetBtn && !lastResetBtn) {
    delay(50); // Simple debounce
    if (currentState == PAUSED || currentState == RESET) {
      currentState = RESET;
      startTime = 0;
      elapsedPausedTime = 0;
      totalElapsedTime = 0;
      Serial.println("Stopwatch: RESET");
    }
  }
  lastResetBtn = resetBtn;
  
  // 3. Calculate Time
  if (currentState == RUNNING) {
    totalElapsedTime = elapsedPausedTime + (now - startTime);
  }
  
  // Format MM:SS:CC (Minutes, Seconds, Centiseconds)
  unsigned long ms = totalElapsedTime;
  unsigned long min = (ms / 60000) % 60;
  unsigned long sec = (ms / 1000) % 60;
  unsigned long cs  = (ms / 10) % 100; // Centiseconds (1/100s)
  
  updateDisplay(min, sec, cs);
}

void updateDisplay(int m, int s, int c) {
  display.clearDisplay();
  
  // Header
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("OLED STOPWATCH");
  
  // Display current state label
  display.setCursor(0, 12);
  if (currentState == RUNNING) display.print("Status: RUNNING");
  else if (currentState == PAUSED) display.print("Status: PAUSED ");
  else display.print("Status: RESET  ");
  
  // Display Time digits (size 2 = large readable text)
  display.setTextSize(2);
  display.setCursor(10, 32);
  
  char timeStr[16];
  sprintf(timeStr, "%02d:%02d.%02d", m, s, c);
  display.print(timeStr);
  
  display.display();
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **SSD1306 OLED**, and two **Buttons** onto the canvas.
2. Wire OLED to **GPIO21/GPIO22**, Button A to **GPIO4**, and Button B to **GPIO15**.
3. Paste the code and click **Run**.
4. Click Button A to start the stopwatch. Watch the centiseconds count up. Click Button A again to pause, and click Button B to reset back to zero.

## Expected Output
Serial Monitor:
```
Stopwatch Online.
Stopwatch: RUNNING
Stopwatch: PAUSED
Stopwatch: RESUMED
Stopwatch: RESET
```

OLED Display (running):
```
OLED STOPWATCH
Status: RUNNING
00:14.52
```

## Expected Canvas Behavior
* At boot, the OLED displays "Status: RESET" and "00:00.00".
* Pressing the Start button widget begins counting. The centiseconds spin rapidly.
* Pressing the Start button again pauses the counter.
* Pressing the Reset button widget resets the numbers to zero (only when paused).

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `elapsedPausedTime += (now - startTime)` | Saves current elapsed time segments to buffer when entering pause states. |
| `sprintf(..., "%02d:%02d.%02d")` | Formats clock output segments with leading zeros. |
| `(ms / 10) % 100` | Extracts centiseconds (hundredths of a second) from raw milliseconds. |

## Hardware & Safety Concept: Non-Blocking Timing vs Delay
Using `delay()` halts the processor entirely, preventing it from checking buttons or updating displays during the wait window. High-performance instruments use **non-blocking time tracking** (`millis()` / `micros()`). By storing start timestamps and comparing them against the current system run time, the code calculates precise durations while allowing the loop to query inputs thousands of times per second.

## Try This! (Challenges)
1. **Lap Timer Memory**: Add a lap logging button on GPIO 12. If clicked during running states, save the current time segment and print it.
2. **Alert Buzzer beep**: Add a buzzer that chirps on button clicks and plays a melody when stopwatch exceeds 1 minute.
3. **Countdown Alarm Mode**: Re-wire the inputs to configure a countdown kitchen timer instead of an ascending stopwatch.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Stopwatch counts too slowly | Screen rendering lag | Decrease screen update rate or run I2C at 400kHz |
| Button triggers repeatedly | Bounce noise | Adjust the debounce logic or increase button filter delays |
| Time is not accurate | Millis drift | `millis()` is accurate for general use. For high precision, use DS3231 RTC square wave outputs |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [112 - ESP32 DS3231 Real Time Clock Display](112-esp32-ds3231-real-time-clock-display.md)
- [60 - ESP32 OLED SSD1306 Display Setup](60-esp32-oled-ssd1306-display-setup.md)
- [63 - ESP32 TM1637 Counter Display](63-esp32-tm1637-counter-display.md)
