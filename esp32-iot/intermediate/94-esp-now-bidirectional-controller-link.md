# 94 - ESP-NOW Bidirectional controller link

Configure the ESP32 to establish a two-way, bidirectional ESP-NOW peer link, transmit local potentiometer telemetry on GPIO 34 to a peer node, receive remote potentiometer telemetry from the peer, and adjust a local LED brightness on GPIO 13 via LEDC hardware PWM.

## Goal
Learn how to implement bidirectional ESP-NOW links, register send and receive callbacks, use hardware LEDC PWM channels, and map sensor inputs.

## What You Will Build
An ESP32 DevKitC acts as a bidirectional control node. A potentiometer is on GPIO 34, and an LED on GPIO 13. Every 100 ms, the ESP32 reads its potentiometer and transmits the value over ESP-NOW. Simultaneously, it listens for telemetry from a peer node, maps the received value, and controls its local LED brightness instantly.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Wiper | GPIO34 | Yellow | Analog control input |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Brightness indicator |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the Red LED anode to GPIO 13 via a 330 Ω resistor. Power the potentiometer from the 3.3V rail.

## Code
```cpp
// ESP-NOW Bidirectional controller link (Two-Way Sensor/Actuator Peer Link)
#include <WiFi.h>
#include <esp_now.h>

// Broadcast MAC Address (sends to all listening nodes)
uint8_t peerAddress[] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};

// 1. Define telemetry data packet structure
typedef struct struct_message {
  int potVal;
} struct_message;

struct_message myTxData;
struct_message myRxData;
esp_now_peer_info_t peerInfo;

const int POT_PIN = 34;
const int LED_PIN = 13;

// LEDC PWM settings
const int pwmChannel = 0;
const int pwmFreq = 5000;
const int pwmResolution = 8; // 8-bit resolution: 0-255

// Send Callback
void OnDataSent(const uint8_t *mac_addr, esp_now_send_status_t status) {
  // Send status debug print (optional, can be commented to reduce serial output)
}

// Receive Callback
// 2. Processes incoming peer telemetry and writes LED PWM duty cycles
void OnDataRecv(const uint8_t * mac, const uint8_t *incomingData, int len) {
  memcpy(&myRxData, incomingData, sizeof(myRxData));
  
  // Map remote potentiometer value (0-4095) to 8-bit PWM (0-255)
  int ledBrightness = map(myRxData.potVal, 0, 4095, 0, 255);
  
  // Write to hardware LEDC PWM channel
  ledcWrite(pwmChannel, ledBrightness);
  
  static unsigned long lastLog = 0;
  if (millis() - lastLog >= 500) { // Limit log output frequency
    Serial.printf("[ESP-NOW] RX Peer Pot: %d | LED Duty: %d\n", myRxData.potVal, ledBrightness);
    lastLog = millis();
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(POT_PIN, INPUT);
  
  // 3. Configure hardware LEDC PWM channel
  ledcSetup(pwmChannel, pwmFreq, pwmResolution);
  ledcAttachPin(LED_PIN, pwmChannel);
  ledcWrite(pwmChannel, 0); // Start OFF
  
  Serial.println("\nESP-NOW Bidirectional Node Starting...");
  
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
  memcpy(peerInfo.peer_addr, peerAddress, 6);
  peerInfo.channel = 0;  
  peerInfo.encrypt = false;
  
  if (esp_now_add_peer(&peerInfo) != ESP_OK) {
    Serial.println("Failed to add peer!");
    return;
  }
  Serial.println("Peer registered. Link active.");
}

unsigned long lastTxTime = 0;
const unsigned long TX_INTERVAL_MS = 100; // Transmit at 10 Hz

void loop() {
  unsigned long now = millis();
  
  // 4. Send local potentiometer telemetry periodically (10 Hz)
  if (now - lastTxTime >= TX_INTERVAL_MS) {
    myTxData.potVal = analogRead(POT_PIN);
    
    // Transmit packet over ESP-NOW
    esp_now_send(peerAddress, (uint8_t *) &myTxData, sizeof(myTxData));
    
    lastTxTime = now;
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer**, and **LED** onto the canvas.
2. Wire Potentiometer to **GPIO34** and LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Slide the potentiometer widget. On hardware, pair this with a second node running the same code. Verify that sliding the potentiometer on one board changes the LED brightness on the other board.

## Expected Output
Serial Monitor:
```
ESP-NOW Bidirectional Node Starting...
Device MAC Address: 30:AE:A4:07:0D:64
Peer registered. Link active.
[ESP-NOW] RX Peer Pot: 2048 | LED Duty: 127
[ESP-NOW] RX Peer Pot: 4095 | LED Duty: 255
```

## Expected Canvas Behavior
* Adjusting the potentiometer triggers packet transmissions logged in the Serial Monitor.
* Receiving packets from a peer dynamically updates the local LED widget brightness.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `ledcSetup(...)` | Sets up the LEDC PWM channel frequency and resolution. |
| `ledcAttachPin(...)` | Binds the target GPIO output pin to the LEDC channel. |
| `ledcWrite(...)` | Updates the duty cycle (brightness) of the output LED. |
| `esp_now_send(...)` | Transmits the local potentiometer reading to the registered peer address. |

## Hardware & Safety Concept: Bidirectional Radio Duty Cycles
Bidirectional communications require rapid switching between transmission (TX) and reception (RX) states. If nodes transmit too fast (e.g. 100 Hz), they saturate the wireless channel, causing collision rates to soar and dropping packets. Restricting the transmission interval to **10 Hz (100 ms)** provides smooth control feedback while keeping the channel clear.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display both local and remote potentiometer values.
2. **Double Channel Link**: Add a second LED on GPIO 12 and map its brightness to a separate joystick axis over ESP-NOW.
3. **Emergency Disconnect Chime**: Sound an alarm on a buzzer (GPIO 15) if no packets are received from the peer for 3 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED remains OFF | LEDC pin misconfigured | Confirm that the LED is connected to GPIO 13 |
| Jittery brightness | High collision rates | Verify that the transmission interval is at least 100 ms |
| Packets drop frequently | Channel noise | Ensure both nodes use the same WiFi channel (usually 1 or 6) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [08 - ESP32 PWM LED Fader](../../esp32-basic/beginner/08-esp32-pwm-led-fader.md)
- [90 - ESP-NOW Peer-to-Peer communications](90-esp-now-peer-to-peer-communications.md)
- [95 - ESP-NOW Broadcast warning siren](95-esp-now-broadcast-warning-siren.md) (Next project)
