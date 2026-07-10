# 199 - Pico Liquid Filling Datalogger OLED

Build an automated bottling control station that opens a dispensing valve, measures level percentage, displays details on an OLED, and logs events over Bluetooth.

## Goal
Learn how to monitor analog water level sensors, scan start buttons, control solenoid valve relays, design graphical OLED progress bars, and stream Bluetooth audit reports.

## What You Will Build
An automatic liquid dispensing system with wireless audit logs and OLED HUD:
- **Water Level Sensor (GP26)**: Measures liquid height in the bottle.
- **Dispensing Valve Relay (GP10)**: Actuates a 5V solenoid water valve.
- **Start Button (GP16)**: Starts the filling process.
- **SSD1306 OLED (GP4, GP5)**: Displays filling progress text, a visual filling bar, and valve status.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits wireless fill logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Water Level Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Push Button | `button` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Water Sensor | OUT (Analog) | GP26 | Container depth |
| Start Button | Terminal 1 | GP16 | Start filling trigger |
| Relay Module | IN | GP10 | Solenoid valve control |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int WATER_PIN  = 26;
const int START_PIN  = 16;
const int RELAY_PIN  = 10;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Depth limits (raw ADC)
const int EMPTY_LEVEL = 500;  // Bottle empty limit
const int FULL_LEVEL  = 3000; // Bottle full limit

bool fillingActive = false;
int fillCount = 0;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(WATER_PIN, INPUT);
  pinMode(START_PIN, INPUT_PULLUP); // Active-LOW button
  pinMode(RELAY_PIN, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW); // Start with valve closed

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  resetDispenserState();
  Serial1.println("Bottling Station Datalogger Online.");
}

void loop() {
  int level = analogRead(WATER_PIN);
  int startBtn = digitalRead(START_PIN);

  // 1. Detect start button press
  if (startBtn == LOW && !fillingActive) {
    if (level < EMPTY_LEVEL) {
      fillingActive = true;
      digitalWrite(RELAY_PIN, HIGH); // Open dispensing valve
      
      fillCount = fillCount + 1;
      Serial1.print("EVENT:FILL_START - Cycle #");
      Serial1.println(fillCount);
    } else {
      display.clearDisplay();
      display.setTextColor(SSD1306_WHITE);
      display.setTextSize(1);
      display.setCursor(10, 20);
      display.print("Insert Empty");
      display.setCursor(10, 35);
      display.print("Bottle & Try...");
      display.display();
      delay(1500);
      resetDispenserState();
    }
  }

  // 2. Filling operations
  if (fillingActive) {
    int progress = (level - EMPTY_LEVEL) * 100 / (FULL_LEVEL - EMPTY_LEVEL);
    if (progress < 0) { progress = 0; }
    if (progress > 100) { progress = 100; }

    // Map progress (0-100%) to screen bar width (0-128 pixels)
    int barWidth = progress * 128 / 100;

    display.clearDisplay();

    // Draw header block
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(18, 3);
    display.print("BOTTLING ACTIVE");

    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);
    display.setCursor(10, 20);
    display.print("Filled: ");
    display.print(progress);
    display.print(" %");

    display.setCursor(10, 32);
    display.print("Valve : OPEN/FLOW");

    // Draw progress fill bar
    display.drawRect(0, 46, 128, 14, SSD1306_WHITE);
    if (barWidth > 0) {
      display.fillRect(2, 48, barWidth - 4, 10, SSD1306_WHITE);
    }
    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();

    // Close valve when full
    if (level >= FULL_LEVEL) {
      fillingActive = false;
      digitalWrite(RELAY_PIN, LOW); // Close valve
      
      display.clearDisplay();
      display.setTextColor(SSD1306_WHITE);
      display.setTextSize(1);
      display.setCursor(10, 20);
      display.print("Filling Done!");
      display.setCursor(10, 35);
      display.print("Remove Bottle   ");
      display.display();
      
      Serial1.print("EVENT:FILL_DONE - Cycle #");
      Serial1.println(fillCount);
      delay(3000);
      resetDispenserState();
    }
  }

  delay(100);
}

void resetDispenserState() {
  fillingActive = false;
  digitalWrite(RELAY_PIN, LOW);

  display.clearDisplay();
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(18, 3);
  display.print("BOTTLING STATION");

  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(10, 24);
  display.print("Valve: CLOSED");
  display.setCursor(10, 40);
  display.print("Press Start...");
  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Water Level Sensor** (represented by potentiometer), **Push Button**, **Relay**, **SSD1306 OLED**, and **HC-05** onto the canvas.
2. Connect sensors, relay, screen, and Bluetooth.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the water sensor to empty, click the Start button to open the relay, then slide the sensor to full to close it, and check the OLED progress bar.

## Expected Output

Terminal (Bluetooth):
```
Bottling Station Datalogger Online.
EVENT:FILL_START - Cycle #1
EVENT:FILL_DONE - Cycle #1
```

## Expected Canvas Behavior
* Startup: OLED reads `Press Start...`. Relay is OFF.
* Press Start (empty bottle): Relay turns ON. OLED reads `Filled: 0 %` and draws progress bar. Bluetooth prints `EVENT:FILL_START`.
* Depth reaches maximum (> 3000): Relay turns OFF. OLED reads `Filling Done!`. Bluetooth prints `EVENT:FILL_DONE`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.print("EVENT:FILL_START ...")` | Sends a start-of-cycle audit log over the Bluetooth UART channel to record active filling operations. |

## Hardware & Safety Concept: Dispenser Overflow Prevention
Liquid filling stations must prevent spills and overflows. If the water level sensor fails or gets dirty, it may keep reporting empty, causing the valve to stay open and overflow the bottle. To prevent this, smart systems use an **auto-shutoff timeout** (e.g. closing the valve if filling takes longer than 10 seconds) or add secondary mechanical overflow floats.

## Try This! (Challenges)
1. **Critical Overflow Alarm**: Sound a buzzer on GP14 if the water level exceeds a maximum limit of 3500.
2. **Emergency Stop**: Wire a second button on GP15 that closes the valve immediately and cancels the filling sequence.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Valve closes immediately after opening | Depth sensor calibration issue | Ensure the empty container sensor reading is below `EMPTY_LEVEL` (500) before clicking the start trigger. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [41 - Pico Water Level](../../beginner/41-pico-water-level.md)
- [102 - Pico Water Station](../intermediate/102-pico-water-station.md)
- [138 - Pico Liquid Filling](../advanced/138-pico-liquid-filling.md)
- [183 - Pico Liquid Filling Datalogger](183-pico-liquid-filling-datalogger.md)
