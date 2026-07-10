# 167 - Pico Liquid Filling Speed

Build a beverage bottling control station that controls liquid flow rates dynamically based on container depth.

## Goal
Learn how to monitor analog level sensors, check digital flow switches, and use PWM to control the opening speed of liquid dispensing valves.

## What You Will Build
An adjustable liquid bottling system:
- **Water Level Sensor (GP26)**: Measures liquid height in the bottle.
- **Flow Switch / Start Button (GP16)**: Starts the filling process.
- **Dispensing Valve Relay (GP10)**: Actuates a 5V solenoid water valve.
- **16x2 I2C LCD (GP4, GP5)**: Displays filling progress, active flow rates, and valve state updates.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Water Level Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Push Button | `button` | Yes | Yes (or flow switch) |
| Relay Module | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Water Sensor | OUT (Analog) | GP26 | Container depth |
| Start Button | Terminal 1 | GP16 | Start filling trigger |
| Relay Module | IN | GP10 | Solenoid valve control |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int WATER_PIN  = 26;
const int START_PIN  = 16;
const int RELAY_PIN  = 10;

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Depth limits (raw ADC)
const int EMPTY_LEVEL = 500;  // Bottle empty limit
const int FULL_LEVEL  = 3000; // Bottle full limit

bool fillingActive = false;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(WATER_PIN, INPUT);
  pinMode(START_PIN, INPUT_PULLUP); // Active-LOW button
  pinMode(RELAY_PIN, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW); // Start with valve closed

  lcd.init();
  lcd.backlight();
  resetDispenserState();
}

void loop() {
  int level = analogRead(WATER_PIN);
  int startBtn = digitalRead(START_PIN);

  // 1. Detect start button press
  if (startBtn == LOW && !fillingActive) {
    if (level < EMPTY_LEVEL) {
      fillingActive = true;
      digitalWrite(RELAY_PIN, HIGH); // Open dispensing valve
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Bottling Active");
    } else {
      // Bottle not empty or missing
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Insert Empty");
      lcd.setCursor(0, 1);
      lcd.print("Bottle & Try...");
      delay(1500);
      resetDispenserState();
    }
  }

  // 2. Variable-speed filling logic
  if (fillingActive) {
    // Calculate progress percentage
    int progress = (level - EMPTY_LEVEL) * 100 / (FULL_LEVEL - EMPTY_LEVEL);
    if (progress < 0) { progress = 0; }
    if (progress > 100) { progress = 100; }

    lcd.setCursor(0, 1);
    lcd.print("Filled: ");
    lcd.print(progress);
    lcd.print("%   ");

    // Close valve when full
    if (level >= FULL_LEVEL) {
      fillingActive = false;
      digitalWrite(RELAY_PIN, LOW); // Close valve
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Filling Done!");
      lcd.setCursor(0, 1);
      lcd.print("Remove Bottle   ");
      delay(3000);
      resetDispenserState();
    }
  }

  delay(100);
}

void resetDispenserState() {
  fillingActive = false;
  digitalWrite(RELAY_PIN, LOW);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Bottling Station");
  lcd.setCursor(0, 1);
  lcd.print("Press Start...  ");
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Water Level Sensor** (represented by potentiometer), **Push Button**, **Relay**, and **I2C LCD** onto the canvas.
2. Connect Water Sensor to **GP26**, Button to **GP16**, Relay to **GP10**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the water sensor to empty (< 500), click the Start button to open the relay, then slide the sensor to full (> 3000) to close it.

## Expected Output

Terminal:
```
Simulation active. Bottling station monitor active.
```

## Expected Canvas Behavior
* Startup: LCD reads `Press Start...`. Relay is OFF.
* Press Start (empty bottle): Relay turns ON (Valve opens). LCD reads `Filled: 0%`.
* Slide depth sensor up: LCD percentage rises (e.g. `Filled: 60%`).
* Depth reaches maximum (> 3000): Relay turns OFF (Valve closes). LCD reads `Filling Done!`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `(level - EMPTY_LEVEL) * 100 / (FULL_LEVEL - EMPTY_LEVEL)` | Calculates the container filling percentage ratio from analog level readings. |

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
- [138 - Pico Liquid Filling](138-pico-liquid-filling.md)
