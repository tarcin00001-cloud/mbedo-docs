# 183 - Pico Liquid Filling Datalogger

Build an automated bottling control station that opens a dispensing relay valve, monitors liquid levels, and streams fill cycle logs over Bluetooth.

## Goal
Learn how to monitor analog depth sensors, check digital start button keys, control liquid solenoid valves, and stream wireless audit logs over Bluetooth.

## What You Will Build
An automatic liquid dispensing system with wireless audit logs:
- **Water Level Sensor (GP26)**: Measures liquid height in the bottle.
- **Dispensing Valve Relay (GP10)**: Actuates a 5V solenoid water valve.
- **Start Button (GP16)**: Starts the filling process.
- **16x2 I2C LCD (GP4, GP5)**: Displays filling progress (e.g. "Filled: 60%") and valve status.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits wireless fill logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Water Level Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Push Button | `button` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Water Sensor | OUT (Analog) | GP26 | Container depth |
| Start Button | Terminal 1 | GP16 | Start filling trigger |
| Relay Module | IN | GP10 | Solenoid valve control |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
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

  lcd.init();
  lcd.backlight();
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
      
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Bottling Active");
      
      fillCount = fillCount + 1;
      Serial1.print("EVENT:FILL_START - Cycle #");
      Serial1.println(fillCount);
    } else {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Insert Empty");
      lcd.setCursor(0, 1);
      lcd.print("Bottle & Try...");
      delay(1500);
      resetDispenserState();
    }
  }

  // 2. Filling operations
  if (fillingActive) {
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
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Bottling Station");
  lcd.setCursor(0, 1);
  lcd.print("Press Start...  ");
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Water Level Sensor** (represented by potentiometer), **Push Button**, **Relay**, **I2C LCD**, and **HC-05** onto the canvas.
2. Connect sensors, relay, LCD, and Bluetooth.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the water sensor to empty, click the Start button to open the relay, then slide the sensor to full to close it, and check the logs over Bluetooth.

## Expected Output

Terminal (Bluetooth):
```
Bottling Station Datalogger Online.
EVENT:FILL_START - Cycle #1
EVENT:FILL_DONE - Cycle #1
```

## Expected Canvas Behavior
* Startup: LCD reads `Press Start...`. Relay is OFF.
* Press Start (empty bottle): Relay turns ON. LCD reads `Filled: 0%`. Bluetooth prints `EVENT:FILL_START`.
* Depth reaches maximum (> 3000): Relay turns OFF. LCD reads `Filling Done!`. Bluetooth prints `EVENT:FILL_DONE`.

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
- [167 - Pico Liquid Filling Speed](../advanced/167-pico-liquid-filling-speed.md)
