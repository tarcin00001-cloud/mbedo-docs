# 182 - ESP32 EEPROM Settings Store & Restore

Build a non-volatile user configuration station on the ESP32 that reads a target value from a potentiometer, saves it to emulated EEPROM flash memory when a button is pressed, and restores the value on boot.

## Goal
Learn how to read and write data to non-volatile flash memory, commit changes, and read boot configurations.

## What You Will Build
A potentiometer on GPIO 34 selects a target temperature (20.0 °C to 40.0 °C). A button on GPIO 4 saves the setting. A buzzer is on GPIO 15, and a 16x2 LCD on I2C. On boot, the ESP32 reads the saved temperature and boot counter from EEPROM. Pressing the button saves the current potentiometer value to EEPROM, sounding the buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| 10 kΩ Resistor (for button) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Wiper | GPIO34 | Yellow | Temp threshold adjustment |
| Push Button | Pin 1 / Pin 2 | GPIO4 / 3V3 | Orange / Red | Save trigger input (active-HIGH) |
| Push Button | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Save confirmation beep |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the button with an external 10 kΩ pull-down resistor to GPIO 4. Share the I2C bus lines for the LCD.

## Code
```cpp
// EEPROM Settings Store & Restore (Non-volatile settings save)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <EEPROM.h>

const int POT_PIN = 34;
const int BUTTON_PIN = 4;
const int BUZZER_PIN = 15;

LiquidCrystal_I2C lcd(0x27, 16, 2);

// EEPROM addresses
// Size of allocation in bytes (max 4096)
const int EEPROM_SIZE = 64; 
// 4 bytes for float temperature
const int ADDR_SAVED_TEMP = 10; 
// 4 bytes for integer boot counter
const int ADDR_BOOT_COUNT = 20; 

int bootCount = 0;
float savedTempLimit = 25.0; // Default fallback temperature
float liveTempLimit = 25.0;

void setup() {
  Serial.begin(115200);
  
  pinMode(BUTTON_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  lcd.init();
  lcd.backlight();
  
  // 1. Initialize emulated EEPROM
  Serial.println("Initializing EEPROM...");
  if (!EEPROM.begin(EEPROM_SIZE)) {
    Serial.println("EEPROM initialization failed!");
    lcd.setCursor(0, 0);
    lcd.print("EEPROM ERROR!   ");
    while(1) {}
  }
  
  // 2. Read boot counter from EEPROM
  // If the EEPROM is unwritten, it returns 0xFFFFFFFF or similar (-1); handle fallback
  int storedCount = EEPROM.readInt(ADDR_BOOT_COUNT);
  if (storedCount < 0 || storedCount > 50000) {
    bootCount = 1;
  } else {
    bootCount = storedCount + 1;
  }
  
  // Write updated boot count back to EEPROM
  EEPROM.writeInt(ADDR_BOOT_COUNT, bootCount);
  EEPROM.commit(); // Must call commit to write to physical flash!
  
  // 3. Read saved temperature limit from EEPROM
  float storedTemp = EEPROM.readFloat(ADDR_SAVED_TEMP);
  // Check if unwritten flash (NaN or out of bounds)
  if (isnan(storedTemp) || storedTemp < 10.0 || storedTemp > 50.0) {
    savedTempLimit = 25.0; // Use fallback default
  } else {
    savedTempLimit = storedTemp;
  }
  
  // Display boot telemetry on LCD
  lcd.setCursor(0, 0);
  lcd.print("Boot Count: "); lcd.print(bootCount);
  lcd.setCursor(0, 1);
  lcd.print("Saved T: "); lcd.print(savedTempLimit, 1); lcd.print("C");
  
  Serial.print("Boot Count: "); Serial.println(bootCount);
  Serial.print("Restored Temp Limit: "); Serial.print(savedTempLimit); Serial.println(" C");
  
  delay(2000);
  lcd.clear();
}

void loop() {
  // Read current potentiometer setting (20.0C to 40.0C)
  int rawPot = analogRead(POT_PIN);
  liveTempLimit = 20.0 + (rawPot * 20.0 / 4095.0);
  
  // Read save button state
  bool savePressed = (digitalRead(BUTTON_PIN) == HIGH);
  
  if (savePressed) {
    // 4. Save current live setting to EEPROM
    savedTempLimit = liveTempLimit;
    
    EEPROM.writeFloat(ADDR_SAVED_TEMP, savedTempLimit);
    
    // Commit changes to physical flash memory
    bool success = EEPROM.commit(); 
    
    if (success) {
      Serial.print("Saved new Temp Limit: "); Serial.print(savedTempLimit, 1); Serial.println(" C");
      
      // Update LCD display confirmation
      lcd.setCursor(0, 0);
      lcd.print("SETTINGS SAVED! ");
      lcd.setCursor(0, 1);
      lcd.print("Value: "); lcd.print(savedTempLimit, 1); lcd.print("C   ");
      
      // Sound buzzer chime
      digitalWrite(BUZZER_PIN, HIGH); delay(150);
      digitalWrite(BUZZER_PIN, LOW);
      
      delay(1500); // Hold message
      lcd.clear();
    } else {
      Serial.println("Flash write commit failed!");
    }
  }
  
  // Update normal operation HUD
  lcd.setCursor(0, 0);
  lcd.print("Live Set: "); lcd.print(liveTempLimit, 1); lcd.print("C ");
  lcd.setCursor(0, 1);
  lcd.print("Saved:    "); lcd.print(savedTempLimit, 1); lcd.print("C ");
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer**, **Button**, **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Wire Potentiometer to **GPIO34**, Button to **GPIO4**, Buzzer to **GPIO15**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the potentiometer to set a target (e.g. 32.5 °C). Click the button to save it. Watch the buzzer beep.
5. Click **Stop**, then click **Run** again (simulating power down). Watch the LCD restore the saved value (32.5 °C) on boot.

## Expected Output
Serial Monitor:
```
Initializing EEPROM...
Boot Count: 1
Restored Temp Limit: 25.0 C
Saved new Temp Limit: 32.5 C
(Simulated restart...)
Boot Count: 2
Restored Temp Limit: 32.5 C
```

## Expected Canvas Behavior
* At boot, the LCD shows the boot count and the restored temperature.
* Adjusting the potentiometer updates the `Live Set` value on the LCD.
* Pressing the button widget updates the `Saved` value to match the live target and pulses the buzzer.
* Stopping and restarting the simulation restores the saved value, and the boot counter increments.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `EEPROM.begin(64)` | Initializes the EEPROM library, allocating 64 bytes of emulated flash memory. |
| `EEPROM.readFloat(10)` | Reads 4 bytes from address 10 and decodes them as a float. |
| `EEPROM.writeFloat(10, ...)` | Writes the float value to the RAM buffer at address 10. |
| `EEPROM.commit()` | Writes the RAM buffer to physical flash memory. |

## Hardware & Safety Concept: EEPROM Emulation and Flash Wear
The ESP32 does not contain a physical EEPROM chip. Instead, it emulates EEPROM by dedicating a sector of its flash memory. Flash memory has a write limit (typically **10,000 to 100,000 cycles**). If you write to flash in every execution loop, you will wear out the memory sector in a few hours. To prevent this:
1. Only write when a setting changes or when a button is pressed.
2. Always call `commit()` only when writing is complete, never in every loop execution.

## Try This! (Challenges)
1. **Calibration Offset Log**: Add a calibration offset parameter (float) at address 15, allowing users to save sensor offsets.
2. **Clear configuration button**: Add a button combination that resets the EEPROM sector to defaults on boot.
3. **Advanced settings menu**: Expand the LCD screen to display more configuration pages.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Settings reset on reboot | `commit()` not called | Ensure `EEPROM.commit()` is called after writing to save changes |
| Saved values read as `NaN` | First write to memory | Check for invalid values on boot and use a default fallback value |
| ESP32 crashes on write | Size out of bounds | Verify that the target addresses fit within the allocated `EEPROM_SIZE` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [176 - ESP32 Hardware Watchdog Timer Recovery](176-esp32-hardware-watchdog-timer-recovery.md)
- [136 - ESP32 SD Card Data Logger](136-esp32-sd-card-data-logger.md)
- [183 - ESP32 Dual-core Processing Balancer](183-esp32-dual-core-processing-balancer.md)
