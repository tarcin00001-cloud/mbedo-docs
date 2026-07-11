# 90 - ESP-NOW Peer-to-Peer communications

Configure the ESP32 to initialize the connectionless ESP-NOW wireless protocol, retrieve the local hardware MAC address, register a broadcast target peer address (`FF:FF:FF:FF:FF:FF`), and implement transmission status callbacks.

## Goal
Learn how to initialize Espressif's ESP-NOW protocol, configure MAC address routing networks, define transmission struct buffers, and handle send status callbacks.

## What You Will Build
An ESP32 DevKitC boots, sets its WiFi antenna to Station mode (STA), prints its local hardware MAC address to the Serial Monitor, and initializes ESP-NOW. It registers the broadcast address (`FF:FF:FF:FF:FF:FF`) as a peer. Every 2 seconds, it sends a telemetry message package. A callback logs transmission status (Success or Failure) to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// ESP-NOW Peer-to-Peer communications (ESP-NOW Handshake & Struct Sender)
#include <WiFi.h>
#include <esp_now.h>

// Target Peer MAC Address
// FF:FF:FF:FF:FF:FF is the standard broadcast address (sends to all listening nodes)
uint8_t broadcastAddress[] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};

// 1. Define telemetry data packet structure
// Must match exactly on both sender and receiver nodes
typedef struct struct_message {
  char text[32];
  int count;
  float uptime;
  bool flag;
} struct_message;

struct_message myData;
esp_now_peer_info_t peerInfo;

// 2. Transmission Status Callback Handler
// Triggered automatically when a packet is sent, confirming if the peer received it
void OnDataSent(const uint8_t *mac_addr, esp_now_send_status_t status) {
  Serial.print("[ESP-NOW] Send Status: ");
  if (status == ESP_NOW_SEND_SUCCESS) {
    Serial.println("Delivery Success!");
  } else {
    Serial.println("Delivery Fail! Peer offline or out of range.");
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\nESP-NOW Node Initializing...");
  
  // 3. Set WiFi to Station mode (must be active for ESP-NOW radio channels)
  WiFi.mode(WIFI_STA);
  
  // Print local MAC address (used by other nodes to target this device)
  Serial.print("Local Device MAC Address: ");
  Serial.println(WiFi.macAddress());
  
  // 4. Initialize ESP-NOW protocol
  if (esp_now_init() != ESP_OK) {
    Serial.println("[Error] Failed to initialize ESP-NOW!");
    return;
  }
  Serial.println("ESP-NOW active.");
  
  // Register transmission callback
  esp_now_register_send_cb(OnDataSent);
  
  // 5. Register target peer details
  memcpy(peerInfo.peer_addr, broadcastAddress, 6);
  peerInfo.channel = 0;  // Use active WiFi channel
  peerInfo.encrypt = false; // No encryption for broadcast
  
  // Add peer connection
  if (esp_now_add_peer(&peerInfo) != ESP_OK) {
    Serial.println("Failed to add peer!");
    return;
  }
  Serial.println("Broadcast peer registered.");
}

int packetCounter = 0;

void loop() {
  // Populate telemetry message fields
  strcpy(myData.text, "Hello ESP-NOW!");
  myData.count = packetCounter++;
  myData.uptime = (float)millis() / 1000.0;
  myData.flag = true;
  
  Serial.printf("\n[Sending] Count: %d | Uptime: %.2f s\n", myData.count, myData.uptime);
  
  // 6. Transmit packet buffer to registered peer address
  // Parameters: target mac, data buffer pointer, size of buffer
  esp_err_t result = esp_now_send(broadcastAddress, (uint8_t *) &myData, sizeof(myData));
  
  if (result == ESP_OK) {
    Serial.println("Packet sent successfully.");
  } else {
    Serial.println("Error sending the packet!");
  }
  
  delay(2000); // Send every 2 seconds
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the device's MAC address is displayed.

## Expected Output
Serial Monitor:
```
ESP-NOW Node Initializing...
Local Device MAC Address: 30:AE:A4:07:0D:64
ESP-NOW active.
Broadcast peer registered.

[Sending] Count: 0 | Uptime: 1.00 s
Packet sent successfully.
[ESP-NOW] Send Status: Delivery Success!
```

## Expected Canvas Behavior
* The Serial Monitor logs MAC coordinates and displays outgoing transmission callback statuses.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFi.mode(WIFI_STA)` | Configures the radio subsystem into station mode to enable packet transmission. |
| `esp_now_init()` | Mounts the ESP-NOW networking protocol layer on top of the WiFi stack. |
| `esp_now_register_send_cb(...)` | Registers the callback triggered when a packet delivery is completed. |
| `esp_now_send(...)` | Transmits the data packet directly to the specified MAC address. |

## Hardware & Safety Concept: Connectionless Communication and MAC Filtering
* **Connectionless Links**: Unlike standard WiFi clients (which must perform a handshake and join an SSID), ESP-NOW bypasses the connection protocol, allowing devices to transmit data packets instantly.
* **MAC Filtering**: By default, using the broadcast address `{0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF}` transmits packets to all listening nodes on the channel. To prevent interception, specify the exact MAC address of the receiver.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the local MAC address and send status.
2. **Channel Configuration**: Configure the WiFi channel to a specific frequency (e.g. channel 1) to reduce noise.
3. **Buzzer warning tone**: Sound a warning beep (GPIO 15) if delivery fails.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Initialization fails | WiFi mode inactive | Verify that `WiFi.mode(WIFI_STA)` is called before initializing ESP-NOW |
| Send Status displays Fail | Peer node missing | The broadcast address (`0xFF...`) should succeed, but individual peer MAC addresses require the receiver to be powered ON and on the same channel |
| Sockets drop frequently | Blocking delay in callback | Keep callback handlers short; do not call blocking delay functions inside the callback |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [91 - ESP-NOW Master Transmitter node](91-esp-now-master-transmitter-node.md) (Next project)
- [92 - ESP-NOW Slave Receiver node](92-esp-now-slave-receiver-node.md)
