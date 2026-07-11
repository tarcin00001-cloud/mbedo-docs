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
// WiFi Network Scanner (Scan networks and log details)
#include <WiFi.h>

// Helper to translate encryption type enums to readable strings
const char* getEncryptionTypeString(wifi_auth_mode_t type) {
  switch (type) {
    case WIFI_AUTH_OPEN:            return "Open (Unsecured)";
    case WIFI_AUTH_WEP:             return "WEP (Legacy)";
    case WIFI_AUTH_WPA_PSK:         return "WPA-PSK";
    case WIFI_AUTH_WPA2_PSK:        return "WPA2-PSK (Standard)";
    case WIFI_AUTH_WPA_WPA2_PSK:    return "WPA/WPA2-PSK";
    case WIFI_AUTH_WPA2_ENTERPRISE: return "WPA2-Enterprise";
    case WIFI_AUTH_WPA3_PSK:        return "WPA3-PSK (Secure)";
    default:                        return "UNKNOWN";
  }
}

void performNetworkScan() {
  Serial.println("Starting WiFi scan...");
  
  // 1. Scan for networks
  // scanNetworks(async, show_hidden, passive)
  // Returns number of networks found (negative values indicate errors)
  int numNetworks = WiFi.scanNetworks(false, false, false);
  
  if (numNetworks < 0) {
    Serial.println("WiFi scan failed!");
    return;
  }
  
  Serial.println("Scan complete.");
  Serial.print(numNetworks);
  Serial.println(" networks found:");
  Serial.println("------------------------------------------------------------------");
  Serial.printf("%-3s | %-25s | %-7s | %-7s | %s\n", "Num", "SSID", "RSSI", "Channel", "Security");
  Serial.println("------------------------------------------------------------------");
  
  // 2. Loop through and log details of each network
  for (int i = 0; i < numNetworks; ++i) {
    String ssid = WiFi.SSID(i);
    int rssi = WiFi.RSSI(i);
    int channel = WiFi.channel(i);
    wifi_auth_mode_t encType = WiFi.encryptionType(i);
    
    Serial.printf("%-3d | %-25.25s | %-4d dBm | %-7d | %s\n", 
                  i + 1, 
                  ssid.c_str(), 
                  rssi, 
                  channel, 
                  getEncryptionTypeString(encType));
    delay(10);
  }
  Serial.println("------------------------------------------------------------------\n");
  
  // 3. Clear scan results from memory
  WiFi.scanDelete(); 
}

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
  performNetworkScan();
  
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
