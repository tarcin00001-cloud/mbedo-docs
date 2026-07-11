# 92 - ESP-NOW Slave Receiver node

Configure the ESP32 as an ESP-NOW Slave Receiver node, register a packet reception callback handler, listen for direct peer-to-peer radio frames containing commands, and toggle a local LED on GPIO 13 when a "TOGGLE" command is parsed.

## Goal
Learn how to configure ESP-NOW receiver callbacks, parse incoming byte arrays into structured data, extract command variables, and control local actuators.

## What You Will Build
An ESP32 DevKitC acts as a receiver node. An LED is on GPIO 13. The ESP32 sets its WiFi antenna to Station mode and listens for incoming ESP-NOW packets. When it receives a packet containing the command `TOGGLE` from a Master Transmitter (Project 91), it switches the LED state and logs the event to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Controlled output LED |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the Red LED anode to GPIO 13 via a 330 Ω resistor.

## Code
```cpp
// ESP-NOW Slave Receiver node (Wireless Actuator Toggle Receiver)
#include <WiFi.h>
#include <esp_now.h>

// 1. Define command packet structure
// Must match the transmitter node's struct layout exactly
typedef struct struct_message {
  char command[16];
  unsigned long timestamp;
} struct_message;

struct_message myData;

const int LED_PIN = 13;

// 2. Reception Callback Handler
// Automatically executed when the WiFi antenna receives a packet
void OnDataRecv(const uint8_t * mac, const uint8_t *incomingData, int len) {
  // Copy the incoming byte array into our struct variable
  memcpy(&myData, incomingData, sizeof(myData));
  
  // Log sender MAC address and length
  char macStr[18];
  snprintf(macStr, sizeof(macStr), "%02X:%02X:%02X:%02X:%02X:%02X",
           mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);
  
  Serial.printf("[ESP-NOW] Packet received from: %s | Bytes: %d\n", macStr, len);
  Serial.printf("[Payload] Command: %s | Timestamp: %lu ms\n", myData.command, myData.timestamp);
  
  // 3. Evaluate command payload
  if (strcmp(myData.command, "TOGGLE") == 0) {
    bool nextState = !digitalRead(LED_PIN);
    digitalWrite(LED_PIN, nextState ? HIGH : LOW);
    Serial.printf("[Action] LED toggled to %s.\n", nextState ? "ON" : "OFF");
  } else {
    Serial.println("[Warning] Unrecognized command!");
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start OFF
  
  Serial.println("\nESP-NOW Slave Receiver Starting...");
  
  // Set WiFi to Station mode
  WiFi.mode(WIFI_STA);
  Serial.print("Slave MAC Address: ");
  Serial.println(WiFi.macAddress());
  
  // 4. Initialize ESP-NOW
  if (esp_now_init() != ESP_OK) {
    Serial.println("Error initializing ESP-NOW!");
    return;
  }
  
  // 5. Register packet reception callback
  esp_now_register_recv_cb(OnDataRecv);
  
  Serial.println("Listening for incoming peer commands...");
}

void loop() {
  // Receiver node only processes events asynchronously via the callback.
  // The main loop can yield or perform other tasks.
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED to **GPIO13**.
3. Paste the code and click **Run**.
4. In simulation, since you have one board, this node listens for incoming messages. On hardware, pair this with Project 91.

## Expected Output
Serial Monitor:
```
ESP-NOW Slave Receiver Starting...
Slave MAC Address: 30:AE:A4:07:0D:80
Listening for incoming peer commands...
[ESP-NOW] Packet received from: 30:AE:A4:07:0D:64 | Bytes: 20
[Payload] Command: TOGGLE | Timestamp: 4500 ms
[Action] LED toggled to ON.
```

## Expected Canvas Behavior
* The Red LED widget toggles when a valid packet is received.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFi.mode(WIFI_STA)` | Configures the radio subsystem into station mode to enable packet reception. |
| `esp_now_register_recv_cb(...)` | Registers the callback function to handle incoming packets. |
| `memcpy(...)` | Unpacks the raw byte array back into the struct fields. |
| `strcmp(...)` | Compares the command string with the expected toggle command. |

## Hardware & Safety Concept: Callback Thread Safety and Memory Copies
ESP-NOW reception callbacks run on the WiFi system thread, not the main thread. To prevent memory conflicts or crash loops:
1. **Never block in callbacks**: Avoid calling delays, web queries, or complex processing inside the callback.
2. **Buffer Copying**: Copy the incoming payload data immediately using `memcpy` into a local or global struct, then process or flag it for the main loop to handle.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a checklist showing the last command received.
2. **Dynamic command parsing**: Add a `BLINK` command to blink the LED 3 times when received.
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 15) when the LED state changes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| No packets are received | Channel mismatch | Both sender and receiver nodes must be on the same WiFi channel (the channel is inherited from the WiFi gateway/SSID or set programmatically) |
| Sockets drop frequently | Blocking delay in callback | Verify that the reception callback does not contain `delay()` statements |
| The LED does not toggle | Pin assignment conflict | Verify that the LED anode is connected to GPIO 13 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [90 - ESP-NOW Peer-to-Peer communications](90-esp-now-peer-to-peer-communications.md)
- [91 - ESP-NOW Master Transmitter node](91-esp-now-master-transmitter-node.md)
- [93 - ESP-NOW Multi-Sensor broadcast network](93-esp-now-multi-sensor-broadcast-network.md) (Next project)
