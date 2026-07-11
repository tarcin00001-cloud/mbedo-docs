# 95 - ESP-NOW Broadcast warning siren

Configure a safety-critical ESP-NOW node on the ESP32 that reads a local emergency button on GPIO 13, broadcasts alarm status packets (`alarmActive: true`) directly to peer nodes on trigger events, and sounds an alternating audio warning siren on a local passive buzzer on GPIO 15 upon receiving remote alerts.

## Goal
Learn how to implement broadcast emergency alerts, read button trigger transitions, manage passive buzzer sirens using the `tone()` API, and construct distributed safety systems.

## What You Will Build
An ESP32 DevKitC acts as an emergency alert station. A pushbutton is on GPIO 13, and a passive buzzer on GPIO 15. Pressing the button immediately broadcasts an emergency alert packet to all nodes. When any station on the network receives this packet, it sounds an alternating siren (alternating between 1000 Hz and 1500 Hz) to warn users.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Pushbutton | `button` | Yes | Yes |
| Passive Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Pushbutton | Pin 1 / Pin 2 | GPIO13 / GND | Blue / Black | Emergency trigger button |
| Passive Buzzer | VCC (+) | GPIO15 | Yellow | Siren audio output |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the button directly to GPIO 13 and GND. Connect the buzzer VCC to GPIO 15.

## Code
```cpp
// ESP-NOW Broadcast warning siren (Distributed Emergency Siren Node)
#include <WiFi.h>
#include <esp_now.h>

// Broadcast MAC Address (sends to all listening nodes)
uint8_t broadcastAddress[] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};

// 1. Define emergency data packet structure
typedef struct struct_message {
  bool alarmActive;
} struct_message;

struct_message myTxData;
struct_message myRxData;
esp_now_peer_info_t peerInfo;

const int BUTTON_PIN = 13;
const int BUZZER_PIN = 15;

bool lastButtonState = HIGH;
unsigned long lastDebounceTime = 0;
const unsigned long DEBOUNCE_DELAY_MS = 50;

// Play an alternating emergency siren
void soundSiren() {
  Serial.println("[Siren] Sounding emergency warning!");
  for (int i = 0; i < 4; i++) {
    tone(BUZZER_PIN, 1000); // Low frequency note
    delay(150);
    tone(BUZZER_PIN, 1500); // High frequency note
    delay(150);
  }
  noTone(BUZZER_PIN); // Silence
}

// Send Callback
void OnDataSent(const uint8_t *mac_addr, esp_now_send_status_t status) {
  // Optional send log
}

// Receive Callback
// 2. Triggered when a packet arrives, sounding the siren if active
void OnDataRecv(const uint8_t * mac, const uint8_t *incomingData, int len) {
  memcpy(&myRxData, incomingData, sizeof(myRxData));
  
  char macStr[18];
  snprintf(macStr, sizeof(macStr), "%02X:%02X:%02X:%02X:%02X:%02X",
           mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);
  
  if (myRxData.alarmActive) {
    Serial.printf("[ALERT] Emergency signal received from: %s\n", macStr);
    
    // Play alternating siren
    soundSiren();
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\nESP-NOW Emergency Siren Node Starting...");
  
  // Set WiFi to Station mode
  WiFi.mode(WIFI_STA);
  Serial.print("Device MAC Address: ");
  Serial.println(WiFi.macAddress());
  
  // Initialize ESP-NOW
  if (esp_now_init() != ESP_OK) {
    Serial.println("Error initializing ESP-NOW!");
    return;
  }
  
  // Register callbacks
  esp_now_register_send_cb(OnDataSent);
  esp_now_register_recv_cb(OnDataRecv);
  
  // Register peer
  memcpy(peerInfo.peer_addr, broadcastAddress, 6);
  peerInfo.channel = 0;  
  peerInfo.encrypt = false;
  
  if (esp_now_add_peer(&peerInfo) != ESP_OK) {
    Serial.println("Failed to add peer!");
    return;
  }
  Serial.println("Ready. Press button to trigger emergency alert.");
}

void loop() {
  // 3. Read and debounce emergency button
  int reading = digitalRead(BUTTON_PIN);
  
  if (reading != lastButtonState) {
    lastDebounceTime = millis();
  }
  
  if ((millis() - lastDebounceTime) > DEBOUNCE_DELAY_MS) {
    if (reading == LOW && lastButtonState == HIGH) {
      Serial.println("[Trigger] Local emergency button pressed! Broadcasting alert...");
      
      // Populate and send emergency payload
      myTxData.alarmActive = true;
      esp_err_t result = esp_now_send(broadcastAddress, (uint8_t *) &myTxData, sizeof(myData));
      
      if (result == ESP_OK) {
        Serial.println("Alert broadcast successfully.");
      } else {
        Serial.println("Failed to broadcast alert!");
      }
      
      // Also sound local siren to alert local users
      soundSiren();
    }
  }
  
  lastButtonState = reading;
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Pushbutton**, and **Buzzer** onto the canvas.
2. Wire Button to **GPIO13** and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Click the button widget. Verify that the buzzer sounds its siren and the alert logs to the Serial Monitor.

## Expected Output
Serial Monitor:
```
ESP-NOW Emergency Siren Node Starting...
Device MAC Address: 30:AE:A4:07:0D:64
Ready. Press button to trigger emergency alert.
[Trigger] Local emergency button pressed! Broadcasting alert...
Alert broadcast successfully.
[Siren] Sounding emergency warning!
```

## Expected Canvas Behavior
* Clicking the button widget sounds the local buzzer widget (pulsing green) and logs events to the Serial Monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `tone(BUZZER_PIN, ...)` | Generates alternating frequencies on the buzzer pin to sound the siren. |
| `esp_now_send(...)` | Broadcasts the alarm packet to all peers on the channel. |
| `OnDataRecv(...)` | Triggers the local siren when an alarm packet is received from a peer. |
| `noTone(BUZZER_PIN)` | Silences the buzzer after the siren ends. |

## Hardware & Safety Concept: Distributed Safety Alarms and Network Fail-Safes
In industrial plants, safety warnings must reach all operator stations instantly. Traditional network alarms rely on centralized servers that can fail. By using **ESP-NOW connectionless broadcasting**, nodes communicate directly, ensuring alarms are delivered even if routers fail.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a warning message when the alarm is active.
2. **Add Warning Strobe**: Add an LED on GPIO 12 to blink rapidly when the alarm sounds.
3. **Mute Button**: Add code to allow pressing the button to silence a received alarm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound | Buzzer pin misconfigured | Confirm that the passive buzzer is connected to GPIO 15 |
| Double trigger on a single press | Debounce delay too short | Increase the `DEBOUNCE_DELAY_MS` value (e.g. from 50 to 100 ms) |
| Alarm packets are ignored | Callback not registered | Ensure `esp_now_register_recv_cb` is called in setup |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [82 - ESP32 MQTT Subscribe Buzzer melody player](82-esp32-mqtt-subscribe-buzzer-melody-player.md)
- [90 - ESP-NOW Peer-to-Peer communications](90-esp-now-peer-to-peer-communications.md)
- [91 - ESP-NOW Master Transmitter node](91-esp-now-master-transmitter-node.md)
