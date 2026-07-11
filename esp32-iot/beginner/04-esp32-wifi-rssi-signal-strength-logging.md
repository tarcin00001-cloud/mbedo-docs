# 04 - ESP32 WiFi RSSI Signal Strength Logging

Build a wireless signal quality analyzer on the ESP32 that reads the Received Signal Strength Indicator (RSSI) in dBm, converts it to a signal quality percentage (0% to 100%), and logs the values to the Serial Monitor.

## Goal
Learn how to read wireless signal strength (RSSI), calculate signal quality percentages, and diagnose RF interference.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Once connected, it polls the wireless controller every 2 seconds to read the RSSI value. It maps this value to a percentage (0% to 100%) and prints it with a signal health description.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// WiFi RSSI Signal Strength Logging (dBm -> Quality %)
#include <WiFi.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Helper function to map RSSI dBm values to a quality percentage
// RSSI range is typically -50 dBm (Excellent) to -100 dBm (No signal)
int getSignalQuality(int rssi) {
  int quality = 0;
  
  if (rssi <= -100) {
    quality = 0;
  } else if (rssi >= -50) {
    quality = 100;
  } else {
    // Linear map: -100 dBm is 0%, -50 dBm is 100%
    // formula: quality = 2 * (rssi + 100)
    quality = 2 * (rssi + 100);
  }
  return quality;
}

// Helper to describe signal level
const char* getSignalLabel(int quality) {
  if (quality >= 80) return "Excellent (Strong Signal)";
  if (quality >= 60) return "Good (Stable Connection)";
  if (quality >= 40) return "Fair (Acceptable, minor lag)";
  if (quality >= 20) return "Weak (Prone to drops)";
  return "Very Weak (Unstable / Disconnected)";
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\nWiFi RSSI Signal Analyzer");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConnected successfully!");
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    // 1. Read RSSI in dBm
    int rssi = WiFi.RSSI();
    
    // 2. Map to quality percentage
    int quality = getSignalQuality(rssi);
    
    // 3. Get descriptive label
    const char* label = getSignalLabel(quality);
    
    Serial.print("WiFi RSSI: ");
    Serial.print(rssi);
    Serial.print(" dBm | Quality: ");
    Serial.print(quality);
    Serial.print("% | Health: ");
    Serial.println(label);
  } else {
    Serial.println("Network offline!");
  }
  
  delay(2000); // Check every 2 seconds
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Watch the RSSI logs print.
4. On hardware, relocate the ESP32 further from the router. Watch the RSSI value decrease.

## Expected Output
Serial Monitor:
```
WiFi RSSI Signal Analyzer
....
Connected successfully!
WiFi RSSI: -57 dBm | Quality: 86% | Health: Excellent (Strong Signal)
WiFi RSSI: -63 dBm | Quality: 74% | Health: Good (Stable Connection)
WiFi RSSI: -72 dBm | Quality: 56% | Health: Fair (Acceptable, minor lag)
```

## Expected Canvas Behavior
* The Serial Monitor logs the signal parameters (dBm, percentage, and label) every 2 seconds.

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `WiFi.RSSI()` | Reads the active signal strength in dBm. |
| `2 * (rssi + 100)` | Maps the RSSI value to a percentage range (0% to 100%). |
| `getSignalLabel(...)` | Returns a descriptive label of the connection quality based on the calculated percentage. |

## Hardware & Safety Concept: Understanding RSSI dBm Values
RSSI (Received Signal Strength Indicator) is measured in decibels relative to one milliwatt (dBm). Because RF signals drop rapidly with distance, RSSI is always expressed as a negative number:
1. **Excellent (\(\ge -50\) dBm)**: Very strong signal; router is close.
2. **Good (\(-60\) to \(-50\) dBm)**: Strong, stable connection.
3. **Fair (\(-70\) to \(-60\) dBm)**: Acceptable for basic tasks, but prone to interference.
4. **Weak (\(\le -80\) dBm)**: Unstable connection; high packet loss.
Monitoring RSSI helps identify poor antenna placement or RF interference.

## Try This! (Challenges)
1. **Interactive OLED Signal Meter**: Add an OLED screen (Project 60) and display a 5-bar signal strength indicator.
2. **Low Signal Alarm**: Sound a buzzer (GPIO 15) if the signal quality drops below 30%, warning the user of potential disconnection.
3. **RSSI-based sleep adjustment**: Enter light sleep mode if the signal strength is weak, to conserve battery power.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| RSSI always returns 0 | Driver initialization failed | Confirm that the ESP32 is connected to the network before reading RSSI |
| Signal drops near metal | RF absorption | Keep metal plates and shielding away from the PCB antenna |
| Connection drops at -85 dBm | Signal too weak | Relocate the ESP32 closer to the access point or use an external antenna |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [01 - ESP32 WiFi Station Connection](01-esp32-wifi-station-connection.md)
- [06 - ESP32 WiFi Network Scanner](06-esp32-wifi-network-scanner.md)
- [15 - ESP32 WiFi Signal Alert buzzer](15-esp32-wifi-signal-alert-buzzer.md)
