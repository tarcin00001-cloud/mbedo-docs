# 126 - Pico Home Automation

Build a wireless smart home automation hub that controls lighting relays and logs room temperatures via Bluetooth.

## Goal
Learn how to parse complex Bluetooth command strings (UART), actuate relays, read digital temperatures (DHT22), and report system status to both LCD and Bluetooth serial monitors.

## What You Will Build
A smart home gateway hub:
- **HC-05 Bluetooth Module (GP0, GP1)**: Receives commands and transmits logs.
- **Relay Module (GP10)**: Toggles living room lighting.
- **DHT22 Climate Sensor (GP12)**: Reads room temperature.
- **16x2 I2C LCD (GP4, GP5)**: Displays device states (e.g. "Light: ON | T: 24C").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| Relay Module | `relay` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| Relay Module | IN | GP10 | Lighting control |
| DHT22 | SDA | GP12 | Climate sensor |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen |
| All Modules | GND | GND | Shared ground reference |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <LiquidCrystal_I2C.h>

const int RELAY_PIN = 10;
const int DHT_PIN   = 12;
#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

bool lightState = false;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART
  dht.begin();
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start light OFF

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Home Hub");
  lcd.setCursor(0, 1);
  lcd.print("System Ready    ");
  
  Serial1.println("Bluetooth Home Hub Connected.");
  Serial1.println("Commands: 1 (Light ON), 0 (Light OFF), T (Read Temp)");
}

void loop() {
  float temp = dht.readTemperature();

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
    else if (cmd == 'T') {
      if (!isnan(temp)) {
        Serial1.print("Current Temp: ");
        Serial1.print(temp);
        Serial1.println(" C");
      } else {
        Serial1.println("Error: Sensor offline!");
      }
    }
  }

  // Update status screen (avoiding fast flicker)
  lcd.setCursor(0, 1);
  if (lightState) {
    lcd.print("L: ON ");
  } else {
    lcd.print("L: OFF");
  }

  lcd.print(" | Temp:");
  if (!isnan(temp)) {
    lcd.print((int)temp);
    lcd.print("C  ");
  } else {
    lcd.print("--C  ");
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-05**, **Relay**, **DHT22**, and **I2C LCD** onto the canvas.
2. Connect HC-05 to **GP0/GP1**, Relay to **GP10**, DHT22 to **GP12**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Type `'1'` in the Bluetooth terminal to click the relay, and `'T'` to request temperature logs.

## Expected Output

Terminal (Bluetooth):
```
Bluetooth Home Hub Connected.
Light toggled: ON
Current Temp: 24.50 C
```

## Expected Canvas Behavior
* Press `'1'` on BT: Relay closes, LCD row 1 reads `L: ON  | Temp:24C`.
* Press `'0'` on BT: Relay opens, LCD row 1 reads `L: OFF | Temp:24C`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.print(temp)` | Transmits raw temperature values wirelessly over the UART channel to the remote Bluetooth terminal. |

## Hardware & Safety Concept: Bidirectional Telemetry
Smart home gateways are bidirectional: they receive control commands (such as lighting toggles) and transmit sensor telemetry (such as room temperature) back to the user. To prevent transmission errors, real smart home platforms wrap serial streams in structured packages (like JSON or MQTT topics) containing validation checksums.

## Try This! (Challenges)
1. **Critical High Alarm**: Sound a buzzer on GP14 if the temperature exceeds 35°C, and send a "WARNING: OVERHEAT!" message over Bluetooth.
2. **Auto Light Timeout**: Automatically turn the light relay OFF after 10 seconds of inactivity.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Commands do not register | Baud rate mismatch | Ensure the smartphone Bluetooth terminal app is set to 9600 baud rate to match `Serial1.begin(9600)`. |

## Mode Notes
This multi-device I2C and UART project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [73 - Pico DHT LCD](../intermediate/73-pico-dht-lcd.md)
- [92 - Pico Bluetooth Relay](../intermediate/92-pico-bt-relay.md)
