# 151 - Pico Home Automation OLED

Build a wireless smart home gateway hub that controls lighting relays and displays real-time DHT22 climate telemetry on a graphical OLED screen.

## Goal
Learn how to parse wireless serial Bluetooth command characters (UART), actuate power relays, and display climate readouts on SSD1306 OLED displays.

## What You Will Build
A smart home gateway HUD:
- **HC-05 Bluetooth Module (GP0, GP1)**: Receives commands.
- **Relay Module (GP10)**: Toggles living room lighting.
- **DHT22 Climate Sensor (GP12)**: Measures temperature and humidity.
- **SSD1306 OLED (GP4, GP5)**: Displays the current light switch state and climate telemetry.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| Relay Module | `relay` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 lines |
| Relay Module | IN | GP10 | Light switch line |
| DHT22 | SDA | GP12 | Climate sensor (requires 10k pull-up) |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int RELAY_PIN = 10;
const int DHT_PIN   = 12;
#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

bool lightState = false;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART
  dht.begin();
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start light OFF

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  Serial1.println("OLED Home Hub Online.");
}

void loop() {
  float temp = dht.readTemperature();
  float humid = dht.readHumidity();

  // Process Bluetooth commands
  if (Serial1.available()) {
    char cmd = Serial1.read();

    if (cmd == '1') {
      lightState = true;
      digitalWrite(RELAY_PIN, HIGH);
      Serial1.println("Light toggled: ON");
    } 
    else if (cmd == '0') {
      lightState = false;
      digitalWrite(RELAY_PIN, LOW);
      Serial1.println("Light toggled: OFF");
    }
  }

  if (!isnan(temp) && !isnan(humid)) {
    display.clearDisplay();

    // Draw header bar
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(15, 3);
    display.print("SMART HOME HUD");

    // Display values
    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);

    // Row 1: Light status
    display.setCursor(10, 20);
    display.print("Light: ");
    if (lightState) {
      display.print("ON");
    } else {
      display.print("OFF");
    }

    // Row 2: Climate readouts
    display.setCursor(10, 34);
    display.print("Temp : ");
    display.print(temp, 1);
    display.print(" C");

    display.setCursor(10, 48);
    display.print("Humid: ");
    display.print(humid, 1);
    display.print(" %");

    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-05**, **Relay**, **DHT22**, and **SSD1306 OLED** onto the canvas.
2. Connect HC-05 to **GP0/GP1**, Relay to **GP10**, DHT22 to **GP12**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Type `'1'` or `'0'` in the simulated Bluetooth terminal to control the light relay and watch the OLED update.

## Expected Output

Terminal (Bluetooth):
```
OLED Home Hub Online.
Light toggled: ON
```

## Expected Canvas Behavior
* Press `'1'` on BT: Relay closes. OLED reads `Light: ON`, showing temperature and humidity.
* Press `'0'` on BT: Relay opens. OLED reads `Light: OFF`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `display.print("Light: ON")` | Updates the status screen with the active state of the living room lighting. |

## Hardware & Safety Concept: Bidirectional Smart Terminals
Smart home gateways are bidirectional: they receive control commands (such as lighting toggles) and transmit sensor telemetry (such as room temperature) back to the user. To prevent transmission errors, real smart home platforms wrap serial streams in structured packages (like JSON or MQTT topics) containing validation checksums.

## Try This! (Challenges)
1. **Critical High Alarm**: Sound a buzzer on GP14 if the temperature exceeds 35°C, and send a warning message over Bluetooth.
2. **Auto Light Timeout**: Automatically turn the light relay OFF after 10 seconds of inactivity to save energy.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Commands do not register | Baud rate mismatch | Ensure the smartphone Bluetooth terminal app is set to 9600 baud rate to match `Serial1.begin(9600)`. |

## Mode Notes
This multi-device I2C and UART project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [74 - Pico DHT OLED](../intermediate/74-pico-dht-oled.md)
- [92 - Pico Bluetooth Relay](../intermediate/92-pico-bt-relay.md)
- [126 - Pico Home Automation](126-pico-home-automation.md)
