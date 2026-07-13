# 02 - ESP32 WiFi Connection Status Log

Build a network diagnostics logger on the ESP32 that translates the internal WiFi state machine values (`wl_status_t` enums) into readable strings, logging them to the Serial Monitor as the connection proceeds.

## Goal
Learn how to track the internal states of the ESP32 WiFi library, translate enum status codes to readable text, and debug connection processes.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Instead of a simple wait loop, the system samples the network connection state every 200 ms, translates the status code (Idle, Connected, Lost, or Disconnected) to text, and prints it, showing the exact steps of the connection process.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// WiFi Connection Status Log (Print state machine status numbers)
#include <WiFi.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n==================================");
  Serial.println("WiFi Connection Status Diagnostic");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  
  Serial.print("Initial State: ");
  Serial.println(WiFi.status()); // Prints status code (e.g. 6 = WL_DISCONNECTED)
  
  Serial.print("Starting connection to "); 
  Serial.println(ssid);
  WiFi.begin(ssid, password);
}

void loop() {
  Serial.print("Connection State: ");
  Serial.println(WiFi.status());
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.print("Assigned IP: ");
    Serial.println(WiFi.localIP());
    delay(5000); // Reduce log frequency once connected
  } else {
    delay(1000); // Check status every second while connecting
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the system logs status changes as it connects.

## Expected Output
Serial Monitor:
```
==================================
WiFi Connection Status Diagnostic
==================================
Initial State: WL_DISCONNECTED (6: Separated from network)
Starting connection to Wokwi-GUEST
Connection Progress -> State: WL_IDLE_STATUS (0: Temporary idle state)
Connection Progress -> State: WL_CONNECTED (3: Connected successful)
----------------------------------
Final Connection State: WL_CONNECTED (3: Connected successful)
Assigned IP: 10.10.0.3
----------------------------------
```

## Expected Canvas Behavior
* The Serial Monitor prints the connection state machine changes in real time, shifting from `WL_DISCONNECTED` to `WL_IDLE_STATUS` and finally to `WL_CONNECTED`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFi.status()` | Reads the active state code of the Espressif WiFi driver stack. |
| `getWiFiStatusString(...)` | Matches the enum code to a descriptive string for diagnostic logging. |
| `currentStatus != lastStatus` | Ensures status updates are logged only when the connection state changes, preventing spam. |

## Hardware & Safety Concept: WiFi State Machine
The ESP32 WiFi controller runs an internal state machine handled by the Espressif SDK (non-blocking). When `WiFi.begin()` is called, the radio hardware scans for the target SSID, negotiates encryption parameters, verifies passwords, and requests an IP address via DHCP. Monitoring these states allows identifying specific issues, such as credential errors (`WL_CONNECT_FAILED`) or out-of-range APs (`WL_NO_SSID_AVAIL`).

## Try This! (Challenges)
1. **Interactive OLED Debugger**: Add an OLED screen (Project 60) and display the active WiFi status string in real time.
2. **Auto-diagnose and recovery**: If the state changes to `WL_NO_SSID_AVAIL` three times, print a suggestion to relocate the antenna.
3. **Buzzer Error chime**: Sound a distinct double beep if the connection state changes to `WL_CONNECT_FAILED`.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Logs show `WL_NO_SSID_AVAIL` | Target router is offline | Check that the SSID name matches the router broadcast settings |
| State stays stuck in `WL_DISCONNECTED` | WiFi mode incorrect | Ensure `WiFi.mode(WIFI_STA)` is called before `WiFi.begin()` |
| Logs show `WL_CONNECT_FAILED` | Password mismatch | Verify the password characters, ensuring they match the router credentials |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [01 - ESP32 WiFi Station Connection](01-esp32-wifi-station-connection.md)
- [03 - ESP32 WiFi IP Address Logger](03-esp32-wifi-ip-address-logger.md)
- [05 - ESP32 WiFi Disconnect and Polling Auto-reconnect routine](05-esp32-wifi-disconnect-and-auto-reconnect-routine.md)
