# 191 - Pico Cold Chain Logger OLED

Build an advanced vaccine carrier cold chain monitor that draws dual graphical thermometer bars on an OLED and streams CSV logs over Bluetooth.

## Goal
Learn how to interface multiple One-Wire digital temperature probes (DS18B20), draw horizontal comparison scale bars on SSD1306 displays, control coolers, and stream Bluetooth CSV log packets.

## What You Will Build
A graphical cold chain carrier diagnostic station:
- **DS18B20 Probes #1 & #2 (GP12 One-Wire DQ)**: Monitors vaccine storage zones.
- **Relay Module (GP10)**: Toggles a cooling fan or Peltier pump.
- **SSD1306 OLED (GP4, GP5)**: Draws two horizontal thermometer bar gauges showing relative temperature levels, and prints status labels.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits wireless weather logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DS18B20 Temperature Probes | `ds18b20` | Yes (two probes) | Yes (two waterproof DS18B20 sensors) |
| Relay Module | `relay` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| 4.7k-ohm Resistor | `resistor` | Optional | Yes (One-wire pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DS18B20 #1 / #2 | DQ (Data) | GP12 | Shared One-Wire DQ bus (requires 4.7k pull-up) |
| Relay Module | IN | GP10 | Cooling pump control |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int ONE_WIRE_BUS = 12;
const int RELAY_PIN    = 10;

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Safety temp threshold (e.g. standard vaccine storage limit 8.0°C)
const float TEMP_LIMIT = 8.0;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART
  sensors.begin();

  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Cooler OFF

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  // Print CSV Header over Bluetooth
  Serial1.println("Temp1_C,Temp2_C,Cooler_ON");
  delay(1000);
}

void loop() {
  sensors.requestTemperatures();
  
  float t1 = sensors.getTempCByIndex(0);
  float t2 = sensors.getTempCByIndex(1);

  bool coolerActive = false;

  // Check if either probe exceeds safety limit
  if (t1 != DEVICE_DISCONNECTED_C && t2 != DEVICE_DISCONNECTED_C) {
    if (t1 > TEMP_LIMIT || t2 > TEMP_LIMIT) {
      digitalWrite(RELAY_PIN, HIGH); // Turn cooler ON
      coolerActive = true;
    } else {
      digitalWrite(RELAY_PIN, LOW);  // Turn cooler OFF
    }
  }

  // Map temperatures (0-20 C) to horizontal bar widths (0-60 pixels)
  int barWidth1 = t1 * 60 / 20;
  int barWidth2 = t2 * 60 / 20;

  if (barWidth1 < 0) { barWidth1 = 0; } if (barWidth1 > 60) { barWidth1 = 60; }
  if (barWidth2 < 0) { barWidth2 = 0; } if (barWidth2 > 60) { barWidth2 = 60; }

  // Update OLED display
  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(18, 3);
  display.print("COLD CHAIN GRAPH");

  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);

  // Probe 1 display (Row 1)
  display.setCursor(6, 20);
  display.print("T1:");
  display.print(t1, 1);
  display.print("C");
  // Draw thermometer bar 1
  display.drawRect(55, 18, 64, 12, SSD1306_WHITE);
  if (barWidth1 > 0) {
    display.fillRect(57, 20, barWidth1, 8, SSD1306_WHITE);
  }

  // Probe 2 display (Row 2)
  display.setCursor(6, 36);
  display.print("T2:");
  display.print(t2, 1);
  display.print("C");
  // Draw thermometer bar 2
  display.drawRect(55, 34, 64, 12, SSD1306_WHITE);
  if (barWidth2 > 0) {
    display.fillRect(57, 36, barWidth2, 8, SSD1306_WHITE);
  }

  // Status display (Row 3)
  display.setCursor(6, 52);
  display.print("STATUS: ");
  display.print(coolerActive ? "COOLING ACTIVE" : "SECURE/SAFE");

  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  // Stream CSV logs over Bluetooth
  if (t1 != DEVICE_DISCONNECTED_C && t2 != DEVICE_DISCONNECTED_C) {
    Serial1.print(t1, 2);
    Serial1.print(",");
    Serial1.print(t2, 2);
    Serial1.print(",");
    Serial1.println(coolerActive ? 1 : 0);
  }

  delay(2000); // Update once every 2 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two DS18B20 Probes**, **Relay**, **SSD1306 OLED**, and **HC-05** onto the canvas.
2. Connect sensors, relay, OLED, and Bluetooth.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the temperature on either probe above 8°C, and verify that the cooler relay activates and the OLED graph updates.

## Expected Output

Terminal (Bluetooth):
```
Temp1_C,Temp2_C,Cooler_ON
4.50,4.20,0
9.20,4.20,1
```

## Expected Canvas Behavior
* Normal state (T < 8.0 C): OLED reads `STATUS: SECURE/SAFE`. Relay is OFF.
* Temperature warning (> 8.0 C): OLED reads `STATUS: COOLING ACTIVE`. Relay turns ON. Thermometer scale bars fill up.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `t1 * 60 / 20` | Scales the temperature (0 to 20°C) to fit within the 60-pixel width of the horizontal thermometer bar on the OLED. |

## Hardware & Safety Concept: Cold Chain Thermal Storage
Vaccines and pharmaceutical products are highly sensitive to temperature variations. Cold chain transport boxes use vacuum-insulated panels and phase-change materials (PCM) to keep temperatures between 2°C and 8°C. Active dataloggers run continuously to record and upload temperature data, verifying that the cargo stayed within safe limits throughout transport.

## Try This! (Challenges)
1. **Under-Temperature Alarm**: Connect a buzzer on GP14 and sound an alarm if the temperature drops below 2.0°C.
2. **Display Sleep**: Turn OFF the OLED screen automatically if no alarms are active to save battery.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temperature reads extreme defaults | Bus resistor missing | One-Wire DQ data lines require a 4.7k pull-up resistor connected to 3.3V power to transmit data correctly. |

## Mode Notes
This multi-device I2C and One-Wire project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [83 - Pico DS18B20 LCD](../intermediate/83-pico-ds18b20-lcd.md)
- [120 - Pico DS18B20 OLED](../intermediate/120-pico-ds18b20-oled.md)
- [150 - Pico DS18B20 OLED Alarm](../advanced/150-pico-ds18b20-oled-alarm.md)
- [179 - Pico Cold Chain Datalogger](179-pico-cold-chain-datalogger.md)
