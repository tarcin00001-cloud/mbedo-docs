# 109 - ESP32 Analog Scale Reader

Build a digital weighing scale that interfaces a load cell weight sensor via an HX711 24-bit ADC amplifier module, calibrates raw readings, and displays weight measurements on a 16x2 I2C LCD.

## Goal
Learn how to communicate with high-resolution external ADCs (HX711) over a custom serial protocol, tare the scale, apply calibration factors, and display weight in grams.

## What You Will Build
A load cell is connected to an HX711 module, which interfaces with the ESP32 on GPIO 18 (DOUT) and GPIO 19 (SCK). A 16x2 I2C LCD displays the live weight. A button on GPIO 4 tares the scale (offsets it to 0 grams).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Load Cell Sensor (e.g. 5kg) | `load_cell` | Yes | Yes |
| HX711 24-bit ADC Amplifier | `hx711` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Tactile Push Button (Tare) | `button` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HX711 Module | VCC / GND | 5V / GND | Red / Black | Power (5V recommended for analog side) |
| HX711 Module | DOUT | GPIO18 | Yellow | Serial Data Output |
| HX711 Module | PD_SCK | GPIO19 | Green | Power Down & Serial Clock |
| Load Cell | 4 Wires (R, B, W, G) | HX711 Inputs | Red/Black/White/Green | Connect to E+, E-, A-, A+ inputs |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD Power |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock |
| Push Button | Pin 1 / Pin 2 | 3V3 / GPIO4 | Red / Yellow | Tare/Zero button |
| Resistor (10k) | Leg 1 / Leg 2 | GPIO4 / GND | White / Black | Pull-down for tare button |

> **Wiring tip:** Connect the load cell to the HX711 input terminals (E+/E- for power, A+/A- for sensor signal). Wire DOUT to GPIO 18 and SCK to GPIO 19.

## Code
```cpp
// Analog Scale Reader (HX711 + LCD)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include "HX711.h"

const int DOUT_PIN = 18;
const int SCK_PIN = 19;
const int TARE_BTN = 4;

HX711 scale;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Calibration factor (must be adjusted for your specific load cell)
// Set this by placing a known weight (e.g. 100g) on the scale and dividing raw reading by 100
const float CALIBRATION_FACTOR = 420.0; 

void setup() {
  Serial.begin(115200);
  
  pinMode(TARE_BTN, INPUT);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Weighing Scale");
  lcd.setCursor(0, 1);
  lcd.print("Zeroing...");
  
  // Initialise HX711
  scale.begin(DOUT_PIN, SCK_PIN);
  scale.set_scale(CALIBRATION_FACTOR);
  scale.tare(); // Zero the scale on startup
  
  delay(1500);
  lcd.clear();
}

void loop() {
  // Read weight (average of 5 readings for stability)
  float weight = scale.get_units(5); 
  
  // Check if tare button is pressed
  if (digitalRead(TARE_BTN) == HIGH) {
    lcd.setCursor(0, 1);
    lcd.print("Zeroing Scale...");
    Serial.println("Taring Scale...");
    scale.tare();
    delay(500);
    lcd.clear();
  }
  
  // Display Weight on LCD
  lcd.setCursor(0, 0);
  lcd.print("Weight:");
  lcd.setCursor(0, 1);
  lcd.print(weight, 1);
  lcd.print(" g        "); // Trailing spaces
  
  Serial.print("Weight: "); Serial.print(weight, 1); Serial.println(" g");
  
  delay(200); // 5Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **HX711**, **Load Cell**, **Button**, and **16x2 I2C LCD** onto the canvas.
2. Wire DOUT to **GPIO18**, SCK to **GPIO19**, Button to **GPIO4**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Adjust the weight slider on the Load Cell widget. Watch the LCD display grams.
5. Click the button to zero (tare) any current weight.

## Expected Output
Serial Monitor:
```
Weight: 0.0 g
Weight: 154.2 g
Weight: 500.0 g
Taring Scale...
Weight: 0.0 g
```

LCD Display (weighing an object):
```
Weight:
500.0 g
```

## Expected Canvas Behavior
* The LCD displays the weight value.
* Adjusting the simulated weight slider on the load cell widget dynamically changes the weight shown on the LCD.
* Pressing the tare button widget zeros the current weight value.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `scale.begin(DOUT_PIN, SCK_PIN)` | Initialises the digital input pins for communicating with the HX711 chip. |
| `scale.set_scale(CALIBRATION_FACTOR)` | Divides raw ADC counts by this factor to output values calibrated in grams. |
| `scale.tare()` | Resets the zero-offset registry reference to subtract the weight of containers. |
| `scale.get_units(5)` | Reads 5 samples from the 24-bit ADC, averages them, and returns weight. |

## Hardware & Safety Concept: Wheatstone Bridges and 24-bit ADC Amplifiers
Load cells contain strain gauges wired in a **Wheatstone Bridge** configuration. When pressure is applied, the physical deformation changes the strain gauge resistances, creating a tiny differential voltage (measured in microvolts, \(\mu V\)). The ESP32's internal 12-bit ADC is not sensitive enough to read this. The HX711 is a dedicated low-noise, 24-bit analog-to-digital converter (ADC) that amplifies this tiny voltage change so the microcontroller can calculate precise weight.

## Try This! (Challenges)
1. **Libras/Ounces Converter**: Add a button on GPIO 12 that switches the display unit between grams (g) and ounces (oz) (1 oz = 28.35 g).
2. **Calibration Mode helper**: Write code that prints raw readings when a button is held to help calculate the `CALIBRATION_FACTOR` easily.
3. **Weight Limit Alert**: Sound a buzzer if the weight exceeds a safe load capacity limit (e.g. 5000g).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Weight values are negative | Load cell is mounted upside down or wires swapped | Check the arrow on the load cell; flip it or swap the A+/A- signal wires |
| Weight drift | Temperature shifts or mechanical creep | tare the scale between measurements; ensure load cell is mounted securely |
| Reading is always 0.0 | Connection issue between HX711 and ESP32 | Check DOUT and SCK pin configurations |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [58 - ESP32 16×2 I2C LCD Print Text](58-esp32-16x2-i2c-lcd-print-text.md)
- [31 - ESP32 Potentiometer ADC Read](../beginner/31-esp32-potentiometer-adc-read.md)
- [102 - ESP32 Water Level Control Station](102-esp32-water-level-control-station.md)
