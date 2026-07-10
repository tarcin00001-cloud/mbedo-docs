# 164 - Pico Solar Panel HUD

Build a solar charging diagnostics station that measures battery voltage and panel output current to display live power efficiency metrics on an OLED.

## Goal
Learn how to monitor multiple analog inputs, calculate power metrics (voltage, current, power in milliwatts), and design graphical OLED dashboards.

## What You Will Build
A solar power diagnostic console:
- **Voltage Sensor Divider (GP26)**: Measures battery voltage (0 to 15V).
- **ACS712 Current Sensor (GP27)**: Measures charging current (0 to 5A).
- **SSD1306 OLED (GP4, GP5)**: Displays voltage, current, and calculated charging power in milliwatts ($P = V \times I$).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Voltage Sensor Module | `potentiometer` | Yes (represented by potentiometer) | Yes (resistor divider) |
| ACS712 Current Sensor | `potentiometer` | Yes (represented by potentiometer) | Yes (5A ACS712 module) |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Voltage Sensor | OUT | GP26 | Analog input for voltage divider |
| ACS712 Sensor | OUT | GP27 | Analog input for current sensor |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int VOLT_PIN = 26;
const int ACS_PIN  = 27;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(VOLT_PIN, INPUT);
  pinMode(ACS_PIN, INPUT);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
}

void loop() {
  int rawVolt = analogRead(VOLT_PIN);
  int rawACS  = analogRead(ACS_PIN);

  // 1. Convert voltage: maps 12-bit ADC (0-4095) to 0-15.0V range
  float voltage = rawVolt * 15.0 / 4095.0;

  // 2. Convert current: maps 12-bit ADC (0-4095) to 0-5000mA range
  float current_mA = rawACS * 5000.0 / 4095.0;

  // 3. Calculate Power (Power = Voltage * Current)
  // Current in Amps = current_mA / 1000.0
  float power_mW = voltage * current_mA;

  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(18, 3);
  display.print("SOLAR POWER HUD");

  // Display values
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);

  // Row 1: Voltage
  display.setCursor(10, 20);
  display.print("Volts: ");
  display.print(voltage, 2);
  display.print(" V");

  // Row 2: Current
  display.setCursor(10, 34);
  display.print("Amps : ");
  display.print(current_mA / 1000.0, 3);
  display.print(" A");

  // Row 3: Power
  display.setCursor(10, 48);
  display.print("Power: ");
  display.print(power_mW, 1);
  display.print(" mW");

  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  delay(500); // Update twice per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two potentiometers** (for voltage/current), and **SSD1306 OLED** onto the canvas.
2. Connect Voltage pot to **GP26**, Current pot to **GP27**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the potentiometer controls to adjust simulated charging rates and watch the OLED calculate power.

## Expected Output

Terminal:
```
Simulation active. Solar power diagnosis engine online.
```

## Expected Canvas Behavior
* Startup: OLED reads `Volts: 0.00 V` / `Amps: 0.000 A` / `Power: 0.0 mW`.
* Slide Voltage to maximum (GP26 high): OLED reads `Volts: 15.00 V`.
* Slide Current to maximum (GP27 high): OLED reads `Amps: 5.000 A`, and `Power` updates to `75000.0 mW`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `voltage * current_mA` | Multiplies voltage by current in milliamperes to calculate power load in milliwatts. |

## Hardware & Safety Concept: Solar Charge Controller Diagnostics
Diagnostic panels let engineers monitor battery states and panel output in solar charging systems. High voltage or overcurrent can damage batteries, so charge controllers monitor these metrics and automatically open cutoff relays if charging parameters cross safety limits.

## Try This! (Challenges)
1. **Overcharge Latch**: Connect a relay on GP10 and open it to cut power if voltage exceeds 14.2V.
2. **Efficiency Rating**: Add code to calculate and display charging efficiency if you enter a panel reference capacity.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Voltage shows static maximum reading | Divider reference missing | Ensure you are scaling the analog readings correctly in code to map the raw 12-bit ADC value (0-4095) to the correct voltage scale. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [34 - Pico LDR Sensor](../../beginner/34-pico-ldr-sensor.md)
- [134 - Pico Solar Tracker](134-pico-solar-tracker.md)
- [158 - Pico Solar Datalogger](158-pico-solar-datalogger.md)
