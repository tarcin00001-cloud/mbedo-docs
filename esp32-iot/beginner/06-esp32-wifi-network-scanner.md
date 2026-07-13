# 06 - ESP32 WiFi Network Scanner

Build a wireless environment scanner on the ESP32 that sweeps the 2.4 GHz spectrum, detects nearby WiFi access points, and logs their SSID, signal strength (RSSI), channel, and encryption type to the Serial Monitor.

## Goal
Learn how to use the ESP32 WiFi scan library, analyze signal metrics, and determine network encryption types.

## What You Will Build
An ESP32 DevKitC performs a network scan, searching for active wireless networks in range. It prints a detailed table of discovered networks, showing their names, signal strengths, channels, and security settings, and repeats the scan every 10 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// WiFi Network Scanner
#include <WiFi.h>

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\nESP32 WiFi Scanner Initialization");
  
  // Set to Station mode and disconnect from any current network
  WiFi.mode(WIFI_STA);
  WiFi.disconnect();
  delay(100);
}

void loop() {
  Serial.println("Starting WiFi scan...");
  WiFi.scanNetworks();
  
  // Wait 10 seconds before scanning again
  delay(10000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that a list of active simulated networks prints.

## Expected Output
Serial Monitor:
```
ESP32 WiFi Scanner Initialization
Starting WiFi scan...
Scan complete.
2 networks found:
------------------------------------------------------------------
Num | SSID                      | RSSI    | Channel | Security
------------------------------------------------------------------
1   | Wokwi-GUEST               | -55 dBm | 6       | Open (Unsecured)
2   | Home-Network              | -78 dBm | 11      | WPA2-PSK (Standard)
------------------------------------------------------------------
```

## Expected Canvas Behavior
* The Serial Monitor logs a table of discovered networks (name, signal strength, channel, and encryption type) every 10 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFi.scanNetworks(...)` | Runs a network scan, blocking execution until complete and returning the number of networks found. |
| `WiFi.SSID(i)` | Retrieves the SSID name of the discovered network at the specified index. |
| `WiFi.encryptionType(i)` | Reads the security protocol code of the network. |
| `WiFi.scanDelete()` | Deletes the scan results from memory to prevent memory leaks. |

## Hardware & Safety Concept: Spectrum Interference and Wi-Fi Channel Planning
Wi-Fi networks operate in the 2.4 GHz ISM band, which is divided into 14 channels (only channels 1, 6, and 11 are non-overlapping). If multiple routers in the same area use overlapping channels, their signals collide, causing packet loss and slow connections. Performing a network scan allows identifying which channels are congested, helping select the optimal channel for a new router.

## Try This! (Challenges)
1. **OLED Scan HUD**: Add an OLED screen (Project 60) and display the names and strengths of the three strongest networks.
2. **Hidden Network Detector**: Modify `WiFi.scanNetworks` parameters to search for and log hidden networks.
3. **RSSI Signal sort**: Sort the scan results by signal strength (RSSI) in descending order before printing.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Scan returns 0 networks | Antenna shielded | Verify that the ESP32 is in range of an active WiFi router |
| ESP32 crashes during scan | Out of memory | Call `WiFi.scanDelete()` after each scan to release memory |
| Scan fails to start | Mode set incorrectly | Ensure `WiFi.mode(WIFI_STA)` is called before starting the scan |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [01 - ESP32 WiFi Station Connection](01-esp32-wifi-station-connection.md)
- [04 - ESP32 WiFi RSSI Signal strength logging](04-esp32-wifi-rssi-signal-strength-logging.md)
- [07 - ESP32 WiFi Channel scanner](07-esp32-wifi-channel-scanner.md)
