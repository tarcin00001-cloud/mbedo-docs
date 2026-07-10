# 179 - Pico Cold Chain Datalogger

Build a pharmaceutical-grade vaccine cold chain logger that tracks dual DS18B20 probes, display safety statistics on an OLED, and streams audit records over Bluetooth.

## Goal
Learn how to interface multiple digital temp probes (DS18B20) on a One-Wire bus, calculate temperature imbalances, draw stats on OLEDs, and stream wireless telemetry logs over Bluetooth.

## What You Will Build
A vaccine carrier temperature logger:
- **DS18B20 Probes #1 & #2 (GP12 One-WireDQ)**: Monitors vaccine storage zones.
- **Relay Module (GP10)**: Actuates a cooling fan or Peltier cooler.
- **Active Buzzer (GP14)**: Emits warning alarms if temperatures exceed safety bounds.
- **SSD1306 OLED (GP4, GP5)**: Displays current probe temperatures, differentials, and safety status.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits CSV logs wirelessly.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DS18B20 Temperature Probes | `ds18b20` | Yes (two probes) | Yes (two waterproof DS18B20 sensors) |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| 4.7k-ohm Resistor | `resistor` | Optional | Yes (One-wire pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DS18B20 #1 / #2 | DQ (Data) | GP12 | Shared One-Wire DQ bus (requires 4.7k pull-up) |
| Relay Module | IN | GP10 | Cooling pump control |
| Active Buzzer | VCC (+) | GP14 | Warning siren pin |
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
const int BUZZER_PIN   = 14;

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
  pinMode(BUZZER_PIN, OUTPUT);

  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  // Print CSV Header over Bluetooth
  Serial1.println("Temp1_C,Temp2_C,Cooler_ON,Alarm_active");
  delay(1000);
}

void loop() {
  sensors.requestTemperatures();
  
  float t1 = sensors.getTempCByIndex(0);
  float t2 = sensors.getTempCByIndex(1);

  bool coolerActive = false;
  bool alarmActive  = false;

  // Check if either probe exceeds safety limit
  if (t1 != DEVICE_DISCONNECTED_C && t2 != DEVICE_DISCONNECTED_C) {
    if (t1 > TEMP_LIMIT || t2 > TEMP_LIMIT) {
      digitalWrite(RELAY_PIN, HIGH); // Turn cooling pump ON
      coolerActive = true;

      // Pulse alarm buzzer if temp gets critically high (> 12.0 C)
      if (t1 > 12.0 || t2 > 12.0) {
        digitalWrite(BUZZER_PIN, HIGH);
        alarmActive = true;
      } else {
        digitalWrite(BUZZER_PIN, LOW);
      }
    } else {
      digitalWrite(RELAY_PIN, LOW);  // Turn cooler OFF
      digitalWrite(BUZZER_PIN, LOW);
    }
  }

  // Update OLED display
  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(15, 3);
  display.print("COLD CHAIN LOGGER");

  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);

  // Probe 1 display
  display.setCursor(8, 20);
  display.print("P1: ");
  if (t1 != DEVICE_DISCONNECTED_C) {
    display.print(t1, 1);
    display.print(" C");
  } else {
    display.print("ERROR");
  }

  // Probe 2 display
  display.setCursor(8, 34);
  display.print("P2: ");
  if (t2 != DEVICE_DISCONNECTED_C) {
    display.print(t2, 1);
    display.print(" C");
  } else {
    display.print("ERROR");
  }

  // Status display
  display.setCursor(8, 48);
  if (alarmActive) {
    display.print("STATUS: !!! OVERHEAT");
  } else if (coolerActive) {
    display.print("STATUS: COOLING...");
  } else {
    display.print("STATUS: SECURE/SAFE");
  }

  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  // Stream CSV logs over Bluetooth
  if (t1 != DEVICE_DISCONNECTED_C && t2 != DEVICE_DISCONNECTED_C) {
    Serial1.print(t1, 2);
    Serial1.print(",");
    Serial1.print(t2, 2);
    Serial1.print(",");
    Serial1.print(coolerActive ? 1 : 0);
    Serial1.print(",");
    Serial1.println(alarmActive ? 1 : 0);
  }

  delay(2000); // Update once every 2 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two DS18B20 Probes**, **Relay**, **Active Buzzer**, **SSD1306 OLED**, and **HC-05** onto the canvas.
2. Connect sensors and communication lines. Connect OLED/sensors to **GP4/GP5** and **GP12**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the temperature on either probe above 8°C and 12°C, and verify that the cooler relay and alarm buzzer activate.

## Expected Output

Terminal (Bluetooth):
```
Temp1_C,Temp2_C,Cooler_ON,Alarm_active
4.50,4.20,0,0
9.20,4.20,1,0
13.50,4.20,1,1
```

## Expected Canvas Behavior
* Normal state (T < 8.0 C): OLED reads `STATUS: SECURE/SAFE`. Relay and Buzzer are OFF.
* Temperature warning (> 8.0 C): OLED reads `STATUS: COOLING...`. Relay turns ON, buzzer remains OFF.
* Critical over-temp (> 12.0 C): OLED reads `STATUS: !!! OVERHEAT`. Relay turns ON, buzzer sounds continuously.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `t1 > TEMP_LIMIT \|\| t2 > TEMP_LIMIT` | Checks if either storage zone temperature exceeds the maximum limit to activate cooling. |

## Hardware & Safety Concept: Cold Chain Thermal Storage
Vaccines and pharmaceutical products are highly sensitive to temperature variations. Cold chain transport boxes use vacuum-insulated panels and phase-change materials (PCM) to keep temperatures between 2°C and 8°C. Active dataloggers run continuously to record and upload temperature data, verifying that the cargo stayed within safe limits throughout transport.

## Try This! (Challenges)
1. **Under-Temperature Alarm**: Sound a warning if the temperature drops below 2.0°C (frost risk).
2. **Dynamic Log rate**: Stream CSV records faster (every 1 second) when alarms are active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temperature reads extreme defaults | Bus resistor missing | One-Wire DQ data lines require a 4.7k pull-up resistor connected to 3.3V power to transmit data correctly. |

## Mode Notes
This multi-device I2C and One-Wire project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [81 - Pico DS18B20 Serial](../intermediate/81-pico-ds18b20-serial.md)
- [120 - Pico DS18B20 OLED](../intermediate/120-pico-ds18b20-oled.md)
- [150 - Pico DS18B20 OLED Alarm](../advanced/150-pico-ds18b20-oled-alarm.md)
