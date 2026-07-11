# 109 - Analog Scale Reader

Measure physical weight using a load cell connected to an HX711 amplifier, displaying the calibrated weight value in grams on an I2C LCD screen with the VEGA ARIES v3 board.

## Goal
Learn how to interface the 24-bit HX711 ADC weight amplifier, apply scale offsets and calibration scales, and output real-time readings on a character display.

## What You Will Build
A digital bench scale. The system initializes by zeroing the sensor scale (taring). Applying weight on the load cell updates the 16x2 I2C LCD display in real-time, showing the current weight in grams.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HX711 Weight Sensor Module | `hx711` | Yes | Yes |
| Load Cell (0-5kg or similar) | `load_cell` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HX711 Module | VCC | 5V | Red | Amplifier power supply (5V) |
| HX711 Module | GND | GND | Black | Ground reference |
| HX711 Module | DOUT (Data) | GPIO 14 | Yellow | Serial data signal |
| HX711 Module | SCK (Clock) | GPIO 15 | Green | Serial clock signal |
| Load Cell | Red Wire | E+ | Red | Excitation voltage positive |
| Load Cell | Black Wire | E- | Black | Excitation voltage negative |
| Load Cell | White Wire | A- | White | Signal input negative |
| Load Cell | Green Wire | A+ | Green | Signal input positive |
| I2C LCD | VCC | 5V | Red | LCD power supply (5V) |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** The connection between the load cell and the HX711 module uses a wheatstone bridge configuration. Standard load cells have 4 colored wires (Red, Black, White, Green) that must be soldered to E+, E-, A-, and A+ respectively.

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <HX711.h>

const int HX711_DOUT_PIN = 14; // DOUT on GPIO 14
const int HX711_SCK_PIN = 15;  // SCK on GPIO 15

HX711 scale;
LiquidCrystal_I2C lcd(0x27, 16, 2);

float currentWeight = 0.0;
float calibrationFactor = 420.0; // Calibration scale factor
float lastWeight = -1.0;

void setup() {
  Wire.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Scale Ready");

  scale.begin(HX711_DOUT_PIN, HX711_SCK_PIN);
  scale.set_scale(calibrationFactor); // Set calibration factor
  scale.tare();                       // Zero out scale on boot
}

void loop() {
  if (scale.is_ready()) {
    // Read the scale once (using 1 reading to avoid loops inside HX711 library helper)
    float reading = scale.get_units(1);
    if (reading < 0.0) {
      reading = 0.0; // Filter minor negative noise at idle
    }
    currentWeight = reading;
  }

  // Update LCD if weight reading changes
  if (currentWeight != lastWeight) {
    lcd.setCursor(0, 0);
    lcd.print("Weight Reader   ");
    
    lcd.setCursor(0, 1);
    lcd.print("Weight: ");
    lcd.print(currentWeight, 1);
    lcd.print(" g   ");
    
    lastWeight = currentWeight;
  }

  delay(250); // Polling delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **HX711 Weight Sensor**, and **I2C LCD Display** components onto the canvas.
2. Wire the HX711: **VCC** to **5V**, **GND** to **GND**, **DOUT** to **GPIO 14**, and **SCK** to **GPIO 15**. Connect the load cell to HX711 terminals.
3. Wire the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
HX711 Initialized. Taring scale...
Scale Tare Finished.
Weight: 0.0 g
Weight: 145.2 g
```

## Expected Canvas Behavior
* Placing weight on the simulated load cell (or adjusting the weight slider) updates the readout.
* The LCD row 2 changes to reflect the current weight (e.g. `Weight: 145.2 g`).
* Removing the weight updates the display back to `Weight: 0.0 g`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `scale.begin(...)` | Binds the digital data and serial clock lines to the HX711 driver. |
| `scale.set_scale(...)` | Establishes the divisor factor mapping raw counts to grams. |
| `scale.tare()` | Subtracts ambient weight offsets to establish a baseline of zero. |
| `scale.get_units(1)` | Captures a single weight reading from the 24-bit ADC. |

## Hardware & Safety Concept
* **Load Cell Physics**: Load cells contain strain gauges in a Wheatstone bridge circuit. When weight is applied, the metal bar bends slightly, altering the resistance of the strain gauges and producing a microvolt differential voltage output (typically 2mV/V). The HX711 amplifies this tiny signal with 24-bit resolution.
* **Mechanical Stop Protection**: Load cells have maximum weight ratings (e.g., 1kg, 5kg, 10kg). Exceeding this rating can permanently deform the metal, destroying the sensor calibration. In design, mechanical stop guards (e.g., a solid spacer block under the cell) are installed to prevent over-extension.

## Try This! (Challenges)
1. **Tare Reset Button**: Connect a push button on GPIO 16. If pressed, execute `scale.tare()` and display `Taring...` on the LCD for 1 second.
2. **Weight Limit Alarm**: Connect a buzzer on GPIO 12. Sound a fast warning beep if the weight reading exceeds 1000g (overload warning).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Readings display as negative when weight is added | Load cell mounted upside down | Flip the load cell over physically or invert the calibration factor (e.g. `-420.0`). |
| LCD displays `0.0 g` even when load is heavy | Loose DOUT or SCK signal wires | Check that GPIO 14 and 15 are wired properly to DOUT and SCK on the HX711 module. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [58 - 16x2 I2C LCD Hello World](58-16x2-i2c-lcd-hello-world.md)
- [67 - Potentiometer Speed DC Motor (with LCD)](67-potentiometer-speed-dc-motor.md)
