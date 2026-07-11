# 91 - ESP-NOW Master Transmitter node

Configure the ESP32 as an ESP-NOW Master Transmitter node, read a pushbutton trigger on GPIO 13, compile a command data packet structure, and transmit the toggle command to remote receiver nodes over direct peer-to-peer radio frames.

## Goal
Learn how to read button state transitions, define communication structs, register broadcast target peers, and transmit control packets.

## What You Will Build
An ESP32 DevKitC acts as a wireless remote control. A pushbutton is connected to GPIO 13 (using internal pull-up). Pressing the button sends a command payload (`TOGGLE`) over ESP-NOW to all listening receiver nodes on the channel, logging the transmission status to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Pushbutton | `button` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Pushbutton | Pin 1 / Pin 2 | GPIO13 / GND | Blue / Black | Control button trigger |

> **Wiring tip:** Connect the button directly to GPIO 13 and GND. The code configures the internal pull-up resistor.

## Code
```cpp
// ESP-NOW Master Transmitter node (Wireless Remote Button Transmitter)
#include <WiFi.h>
#include <esp_now.h>

// Broadcast MAC Address (sends to all listening nodes)
uint8_t receiverAddress[] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};

// 1. Define command packet structure
// Must match the receiver node's struct layout exactly
typedef struct struct_message {
  char command[16];
  unsigned long timestamp;
} struct_message;

struct_message myData;
esp_now_peer_info_t peerInfo;

const int BUTTON_PIN = 13;
bool lastButtonState = HIGH;
unsigned long lastDebounceTime = 0;
const unsigned long DEBOUNCE_DELAY_MS = 50;

// Transmission callback function
void OnDataSent(const uint8_t *mac_addr, esp_now_send_status_t status) {
  Serial.print("[ESP-NOW] Send Status: ");
  Serial.println(status == ESP_NOW_SEND_SUCCESS ? "Delivered" : "Failed");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  
  Serial.println("\nESP-NOW Master Remote Starting...");
  
  // Set WiFi to Station mode
  WiFi.mode(WIFI_STA);
  Serial.print("Master MAC Address: ");
  Serial.println(WiFi.macAddress());
  
  // Initialize ESP-NOW
  if (esp_now_init() != ESP_OK) {
    Serial.println("Error initializing ESP-NOW!");
    return;
  }
  
  // Register send callback
  esp_now_register_send_cb(OnDataSent);
  
  // Register peer
  memcpy(peerInfo.peer_addr, receiverAddress, 6);
  peerInfo.channel = 0;  
  peerInfo.encrypt = false;
  
  if (esp_now_add_peer(&peerInfo) != ESP_OK) {
    Serial.println("Failed to add peer!");
    return;
  }
  Serial.println("Ready to transmit.");
}

void loop() {
  // 2. Read and debounce pushbutton input
  int reading = digitalRead(BUTTON_PIN);
  
  if (reading != lastButtonState) {
    lastDebounceTime = millis();
  }
  
  if ((millis() - lastDebounceTime) > DEBOUNCE_DELAY_MS) {
    // If the state has changed
    if (reading == LOW && lastButtonState == HIGH) {
      // 3. Populate command struct payload
      strcpy(myData.command, "TOGGLE");
      myData.timestamp = millis();
      
      Serial.printf("[Action] Button pressed. Transmitting command '%s'...\n", myData.command);
      
      // 4. Transmit packet over ESP-NOW
      esp_err_t result = esp_now_send(receiverAddress, (uint8_t *) &myData, sizeof(myData));
      
      if (result == ESP_OK) {
        Serial.println("Packet queued successfully.");
      } else {
        Serial.println("Error sending packet.");
      }
    }
  }
  
  lastButtonState = reading;
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Pushbutton** onto the canvas.
2. Wire Button to **GPIO13**.
3. Paste the code and click **Run**.
4. Click the button widget. Verify that "Transmitting command 'TOGGLE'" logs in the Serial Monitor.

## Expected Output
Serial Monitor:
```
ESP-NOW Master Remote Starting...
Master MAC Address: 30:AE:A4:07:0D:64
Ready to transmit.
[Action] Button pressed. Transmitting command 'TOGGLE'...
Packet queued successfully.
[ESP-NOW] Send Status: Delivered
```

## Expected Canvas Behavior
* Clicking the button widget prints transmission events to the Serial Monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `INPUT_PULLUP` | Configures the button pin with an internal pull-up resistor to prevent floating signals. |
| `strcpy(myData.command, ...)` | Populates the command field within the struct payload. |
| `esp_now_send(...)` | Transmits the data packet directly to the registered peer. |
| `OnDataSent(...)` | Receives delivery confirmations from target receivers. |

## Hardware & Safety Concept: Input Debouncing and Radio Wakeup
Pushbuttons use mechanical contacts that bounce when pressed, generating noise that microcontrollers can interpret as multiple rapid presses. Software debouncing (using `millis()` timers) filters this noise. For battery-powered remotes, put the ESP32 into deep sleep and wake it up using external GPIO interrupts when the button is pressed to conserve battery.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a count of button presses.
2. **Double Button Transmitter**: Add a second button on GPIO 12 to send a `RESET` command.
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 15) when the button is pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Double trigger on a single press | Debounce delay too short | Increase the `DEBOUNCE_DELAY_MS` value (e.g. from 50 to 100 ms) |
| Send Status displays Fail | Peer node offline | Verify that the receiver node is powered ON and listening on the same channel |
| Sockets drop frequently | Blocking delay in callback | Keep callback handlers short; do not call blocking delay functions inside the callback |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [04 - ESP32 Button LED control](../../esp32-basic/beginner/04-esp32-button-led-control.md)
- [90 - ESP-NOW Peer-to-Peer communications](90-esp-now-peer-to-peer-communications.md)
- [92 - ESP-NOW Slave Receiver node](92-esp-now-slave-receiver-node.md) (Next project)
