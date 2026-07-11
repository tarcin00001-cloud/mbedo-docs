# 27 - ESP32 UDP Joysticks Telemetry Stream

Build a high-speed telemetry streaming station on the ESP32 that reads the X/Y analog axes and digital switch status of a joystick module, packages the coordinates into a compact string payload, and streams UDP packets at 20 Hz to port 9000.

## Goal
Learn how to sample multiple analog coordinate inputs, construct compact telemetry data packets, manage high-frequency UDP streaming cycles, and minimize latency.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A dual-axis joystick module is wired to GPIO 34 (X-axis), GPIO 35 (Y-axis), and GPIO 12 (Switch input, active-LOW). The ESP32 samples these values every 50 ms (20 Hz stream rate) and transmits a UDP packet containing the coordinates to `255.255.255.255` on port 9000.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Joystick Module | `joystick` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Joystick | VRX (X-axis) | GPIO34 | Yellow | Horizontal analog input |
| Joystick | VRY (Y-axis) | GPIO35 | Blue | Vertical analog input |
| Joystick | SW (Switch) | GPIO12 | Green | Selector button (active-LOW) |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** Configure GPIO 12 with an internal pull-up resistor (`INPUT_PULLUP`) to read the active-LOW switch input.

## Code
```cpp
// UDP Joysticks telemetry stream (High-speed control link)
#include <WiFi.h>
#include <WiFiUdp.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Joystick Pins
const int JS_X_PIN = 34;
const int JS_Y_PIN = 35;
const int JS_SW_PIN = 12;

// Target receiver details
IPAddress targetIP(255, 255, 255, 255); // Broadcast to all local nodes
const int targetPort = 9000;

WiFiUDP udp;

unsigned long lastStreamTime = 0;
const unsigned long STREAM_INTERVAL_MS = 50; // Stream at 20 Hz (every 50 ms)

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(JS_X_PIN, INPUT);
  pinMode(JS_Y_PIN, INPUT);
  pinMode(JS_SW_PIN, INPUT_PULLUP); // Active-LOW push switch
  
  Serial.println("\n==================================");
  Serial.println("ESP32 UDP Joystick Streamer Setup");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  udp.begin(8888); // Bind local port
  Serial.println("Streaming active at 20 Hz.");
  Serial.println("==================================\n");
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    unsigned long now = millis();
    
    // 1. Maintain 50 ms non-blocking update cycle (20 Hz)
    if (now - lastStreamTime >= STREAM_INTERVAL_MS) {
      // 2. Read Joystick signals
      int xVal = analogRead(JS_X_PIN);
      int yVal = analogRead(JS_Y_PIN);
      bool swVal = (digitalRead(JS_SW_PIN) == LOW); // True if pressed
      
      // 3. Format compact telemetry string payload
      String payload = "X:" + String(xVal) + 
                       ",Y:" + String(yVal) + 
                       ",SW:" + String(swVal ? 1 : 0);
      
      // 4. Begin and transmit UDP packet
      udp.beginPacket(targetIP, targetPort);
      udp.print(payload);
      udp.endPacket();
      
      // Print values to Serial Monitor less frequently to prevent congestion
      static int debugCounter = 0;
      debugCounter = (debugCounter + 1) % 10; // Log at 2 Hz
      
      if (debugCounter == 0) {
        Serial.print("[Joystick Stream] Sent -> ");
        Serial.println(payload);
      }
      
      lastStreamTime = now;
    }
  } else {
    Serial.println("WiFi offline!");
    delay(1000);
  }
  
  delay(2); // Yield to system
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Joystick** onto the canvas.
2. Wire VRX to **GPIO34**, VRY to **GPIO35**, and SW to **GPIO12**.
3. Paste the code and click **Run**.
4. Move the joystick widget cursor. Watch the coordinates update in the Serial Monitor.

## Expected Output
Serial Monitor:
```
==================================
ESP32 UDP Joystick Streamer Setup
==================================
WiFi Connected successfully.
Streaming active at 20 Hz.
==================================

[Joystick Stream] Sent -> X:2048,Y:2048,SW:0
[Joystick Stream] Sent -> X:4095,Y:2048,SW:0
[Joystick Stream] Sent -> X:2048,Y:1200,SW:1
```

## Expected Canvas Behavior
* The Serial Monitor logs the raw coordinate telemetry strings at a rate of 2 Hz.
* Dragging the joystick coordinates changes the X and Y values in the output payload.
* Clicking the joystick button changes the SW value from 0 to 1.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `INPUT_PULLUP` | Enables the internal pull-up resistor on the switch pin (reads HIGH when open, LOW when pressed). |
| `now - lastStreamTime >= 50` | Maintains a 50 ms loop cycle, streaming coordinates at a rate of 20 Hz. |
| `udp.print(payload)` | Writes the compact coordinate string into the packet buffer. |

## Hardware & Safety Concept: Streaming Latency in Remote Control Systems
In remote control systems (such as steering a rover or piloting a drone), the latency between the operator moving the joystick and the vehicle responding must be minimized (typically under 100 ms). **UDP** is ideal for this because it has no handshake or packet retry delays. If a packet is lost, the vehicle ignores the drop and processes the next update 50 ms later.

## Try This! (Challenges)
1. **Interactive OLED Status**: Add an OLED screen (Project 60) and display a 2D crosshair representing the joystick position.
2. **Double button trigger**: Add a secondary action button on GPIO 13 to send a boost command string.
3. **Deadband Filter**: Implement a filter to skip sending packets if the joystick remains centered, reducing network load.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| X and Y readings are swapped | Wiring crossed | Verify that VRX is wired to GPIO 34 and VRY to GPIO 35 |
| Switch reads static 0 | Pull-up missing | Ensure the pin mode is set to `INPUT_PULLUP` in setup |
| Telemetry stream is sluggish | Serial logs in loop | Remove or limit serial log statements inside the high-frequency loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [18 - ESP32 UDP Packet Sender](18-esp32-udp-packet-sender.md)
- [19 - ESP32 UDP Packet Receiver](19-esp32-udp-packet-receiver.md)
- [97 - ESP32 Dual Axis Joystick Servos](../intermediate/97-esp32-dual-axis-joystick-servos.md)
