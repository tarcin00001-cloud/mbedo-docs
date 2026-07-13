# 14 - ESP32 Wireless Indicator LED

Build a non-blocking network status indicator on the ESP32 that blinks an LED in distinct patterns to indicate connection states (fast blink for connecting, slow blink for disconnected, solid ON for connected).

## Goal
Learn how to implement multi-state status indicators, manage non-blocking blinking intervals, and monitor connection states.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An LED is on GPIO 13. While searching/connecting, the LED flashes rapidly (100 ms interval). Once connected, the LED glows solid green. If the connection is lost, the LED blinks slowly (1000 ms interval), warning the user.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Green LED | Anode (+) | GPIO13 via 330 Ω | Green | Status indicator LED |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the LED anode to GPIO 13 via a 330 Ω resistor.

## Code
```cpp
// Wireless indicator LED (Non-blocking status blinker)
#include <WiFi.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int INDICATOR_LED = 13;

// Connection states
enum ConnectionState {
  STATE_DISCONNECTED, // Slow blink (1000 ms)
  STATE_CONNECTING,   // Fast blink (100 ms)
  STATE_CONNECTED     // Solid ON
};

ConnectionState activeState = STATE_DISCONNECTED;

// Non-blocking blink timer parameters
unsigned long lastBlinkTime = 0;
bool ledState = LOW;

void updateIndicator() {
  unsigned long now = millis();
  unsigned long interval = 1000; // Default slow blink
  
  // 1. Determine active blink interval based on state
  switch (activeState) {
    case STATE_CONNECTED:
      digitalWrite(INDICATOR_LED, HIGH); // Solid ON
      return;
      
    case STATE_CONNECTING:
      interval = 100; // Fast blink
      break;
      
    case STATE_DISCONNECTED:
      interval = 1000; // Slow blink
      break;
  }
  
  // 2. Toggle LED non-blockingly
  if (now - lastBlinkTime >= interval) {
    ledState = !ledState;
    digitalWrite(INDICATOR_LED, ledState);
    lastBlinkTime = now;
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(INDICATOR_LED, OUTPUT);
  digitalWrite(INDICATOR_LED, LOW);
  
  Serial.println("\nWireless status indicator active.");
  
  // Start connection
  activeState = STATE_CONNECTING;
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
}

void loop() {
  // 3. Monitor connection state
  wl_status_t status = WiFi.status();
  
  if (status == WL_CONNECTED) {
    activeState = STATE_CONNECTED;
  } else if (status == WL_IDLE_STATUS || status == WL_DISCONNECTED) {
    activeState = STATE_CONNECTING;
  } else {
    activeState = STATE_DISCONNECTED;
  }
  
  // 4. Update the LED indicator pattern
  updateIndicator();
  
  // Print status to Serial Monitor less frequently
  static unsigned long lastLog = 0;
  if (millis() - lastLog >= 2000) {
    Serial.print("WiFi status: ");
    Serial.print(status);
    Serial.print(" | Active State: ");
    Serial.println(activeState == STATE_CONNECTED ? "ONLINE" : (activeState == STATE_CONNECTING ? "CONNECTING" : "OFFLINE"));
    lastLog = millis();
  }
  
  delay(10); // Small loop delay
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Watch the LED flash rapidly during the connection handshake and turn solid green once connected.
5. Toggle the simulated network off. Watch the LED blink slowly.

## Expected Output
Serial Monitor:
```
Wireless status indicator active.
WiFi status: 6 | Active State: CONNECTING
WiFi status: 3 | Active State: ONLINE
WiFi status: 3 | Active State: ONLINE
```

## Expected Canvas Behavior
* At boot, the LED flashes rapidly.
* Once connected, the LED glows solid green.
* Toggling the simulated network off causes the LED to blink slowly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `activeState` | Tracks the connection state (disconnected, connecting, or connected). |
| `interval` | Sets the blink rate (100 ms for connecting, 1000 ms for disconnected). |
| `now - lastBlinkTime` | Toggles the LED pin non-blockingly, keeping the main loop active. |

## Hardware & Safety Concept: Non-Blocking Status Indicators
User interfaces must remain responsive. If the controller uses blocking delays to blink an LED, it freezes the entire program, halting key reads or sensor monitoring. Using **Non-blocking blinking timers** and states, the ESP32 updates the indicator LED in the background while keeping the main execution loop active.

## Try This! (Challenges)
1. **Dynamic warning buzzer**: Add a buzzer (GPIO 15) that chirps when the state changes to `STATE_DISCONNECTED`.
2. **Interactive OLED Status**: Add an OLED screen (Project 60) and display a matching status icon (e.g. searching, connected, offline).
3. **RGB status indicator**: Use an RGB LED (GPIO 12, 13, 14) to display blue for searching, green for connected, and red for offline.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The LED stays ON constantly | State mapping error | Verify that `activeState` changes based on the connection status |
| The LED does not light up | LED polarity reversed | Check that the LED anode is connected to the GPIO and the cathode to ground |
| The blinking is uneven or laggy | Blocking delays in loop | Avoid using `delay()` calls inside the main loop to keep the timing accurate |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [01 - ESP32 WiFi Station Connection](01-esp32-wifi-station-connection.md)
- [05 - ESP32 WiFi Disconnect and Polling Auto-reconnect routine](05-esp32-wifi-disconnect-and-auto-reconnect-routine.md)
- [15 - ESP32 WiFi Signal Alert buzzer](15-esp32-wifi-signal-alert-buzzer.md)
