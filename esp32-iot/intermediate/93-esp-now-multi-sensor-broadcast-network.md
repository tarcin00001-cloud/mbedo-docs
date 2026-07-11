# 93 - ESP-NOW Multi-Sensor broadcast network

Configure the ESP32 as an ESP-NOW Receiver Hub, receive climate telemetry packets from multiple remote sensor transmitter nodes, parse the sender's MAC address to identify the sending node, and display the temperature and humidity values on a 16x2 I2C LCD.

## Goal
Learn how to identify sending nodes using MAC address bytes, unpack float variables from byte payloads, display multiple node updates on an LCD, and manage broadcast networks.

## What You Will Build
An ESP32 DevKitC acts as a receiver hub. A 16x2 I2C LCD is on I2C (GPIO 21/22). The ESP32 listens for incoming ESP-NOW packets containing temperature and humidity data. By reading the last byte of the sender's MAC address, it identifies the sender (e.g. Node A or Node B) and displays the readings on the LCD in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Power the LCD from the 5V Vin rail.

## Code
```cpp
// ESP-NOW Multi-Sensor broadcast network (Multi-Node Telemetry Hub)
#include <WiFi.h>
#include <esp_now.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

// 1. Define climate telemetry packet structure
// Must match the transmitters' struct layouts exactly
typedef struct struct_message {
  float temp;
  float humidity;
} struct_message;

struct_message myData;

// Reception Callback Handler
void OnDataRecv(const uint8_t * mac, const uint8_t *incomingData, int len) {
  // Unpack incoming bytes into struct
  memcpy(&myData, incomingData, sizeof(myData));
  
  // 2. Identify the sender node using the last byte of their MAC address
  String nodeName = "Node ";
  
  // Example MAC assignment mapping
  if (mac[5] == 0x64) {
    nodeName = "Node A";
  } else if (mac[5] == 0x80) {
    nodeName = "Node B";
  } else {
    // Fallback: print last MAC byte in HEX representation
    char hexStr[4];
    sprintf(hexStr, "%02X", mac[5]);
    nodeName += String(hexStr);
  }
  
  Serial.printf("[Hub] Data from %s | Temp: %.1f C | Hum: %.1f %%\n", 
                nodeName.c_str(), myData.temp, myData.humidity);
                
  // 3. Display telemetry values on the 16x2 LCD
  lcd.clear();
  
  // Row 0: Node Identity and Temp
  lcd.setCursor(0, 0);
  lcd.print(nodeName);
  lcd.print(": ");
  lcd.print(myData.temp, 1);
  lcd.print(" C");
  
  // Row 1: Humidity info
  lcd.setCursor(0, 1);
  lcd.print("Humidity: ");
  lcd.print(myData.humidity, 1);
  lcd.print(" %");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize LCD screen
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("ESP-NOW Hub");
  lcd.setCursor(0, 1);
  lcd.print("Initializing...");
  
  Serial.println("\nESP-NOW Receiver Hub Starting...");
  
  // Set WiFi to Station mode
  WiFi.mode(WIFI_STA);
  Serial.print("Hub MAC Address: ");
  Serial.println(WiFi.macAddress());
  
  // Initialize ESP-NOW
  if (esp_now_init() != ESP_OK) {
    Serial.println("Error initializing ESP-NOW!");
    lcd.clear();
    lcd.print("Init Error!");
    return;
  }
  
  // Register packet reception callback
  esp_now_register_recv_cb(OnDataRecv);
  
  lcd.clear();
  lcd.print("Ready for nodes...");
  Serial.println("Waiting for sensor node broadcasts...");
}

void loop() {
  // Hub operates asynchronously via reception callbacks.
  // The main loop can yield or perform other tasks.
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **16x2 I2C LCD** onto the canvas.
2. Wire LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. On hardware, setup two transmitter nodes programed to send float data structs, and watch the LCD switch between their updates.

## Expected Output
Serial Monitor:
```
ESP-NOW Receiver Hub Starting...
Hub MAC Address: 30:AE:A4:07:0D:88
Waiting for sensor node broadcasts...
[Hub] Data from Node A | Temp: 24.5 C | Hum: 45.0 %
[Hub] Data from Node B | Temp: 28.2 C | Hum: 50.5 %
```

LCD Display:
```
Node A: 24.5 C
Humidity: 45.0 %
```

## Expected Canvas Behavior
* The LCD widget prints incoming sensor data values and identifies which node sent them.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `mac[5]` | Accesses the final byte of the sender's MAC address array to identify the node. |
| `memcpy(...)` | Unpacks the raw byte array back into the struct fields. |
| `lcd.clear()` | Wipes the LCD screen before printing new data to prevent overlapping characters. |
| `esp_now_register_recv_cb(...)` | Registers the callback handler for incoming packets. |

## Hardware & Safety Concept: Network Identifiers and Data Struct Collisions
In multi-node ESP-NOW networks, all nodes transmit data on the same radio frequency channel. To prevent data corruption:
1. **Define Struct Layouts**: Ensure all transmitters use the exact same struct layout. If structures differ, the receiver will unpack the bytes into the wrong fields, causing corruption.
2. **Filter Sender Nodes**: Use MAC address validation to discard packets sent by unauthorized nodes.

## Try This! (Challenges)
1. **OLED Status Display**: Upgrade the screen to an SSD1306 OLED (Project 60) and display a graph of temperature changes from both nodes.
2. **Node Outage Alert**: Add code to sound an alarm on a buzzer (GPIO 15) if a node stops transmitting for more than 10 seconds.
3. **SPIFFS logger**: Save the values to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD shows garbage characters | LCD address mismatch | Verify the I2C address in `LiquidCrystal_I2C` matches your LCD module (usually `0x27`) |
| Packets from Node A fail | Node struct mismatch | Confirm that the transmitter node uses the exact same struct layout |
| Values are always 0.0 | Sender transmission failed | Confirm that the transmitter node is powered ON and on the same channel |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [88 - ESP32 I2C LCD MQTT telemetry monitor](88-esp32-i2c-lcd-mqtt-telemetry-monitor.md)
- [90 - ESP-NOW Peer-to-Peer communications](90-esp-now-peer-to-peer-communications.md)
- [94 - ESP-NOW Bidirectional controller link](94-esp-now-bidirectional-controller-link.md) (Next project)
