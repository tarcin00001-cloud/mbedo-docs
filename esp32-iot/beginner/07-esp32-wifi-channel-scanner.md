# 07 - ESP32 WiFi Channel Scanner

Build a channel utilization analyzer on the ESP32 that scans the 2.4 GHz spectrum, counts the number of active access points on each channel, and logs a text-based bar chart to identify the cleanest channel.

## Goal
Learn how to analyze WiFi channel distribution, build text-based histograms, and identify the cleanest channels.

## What You Will Build
An ESP32 DevKitC scans nearby networks and counts how many access points are active on each channel (1 to 13). It displays a text-based bar chart on the Serial Monitor showing channel usage and suggests the cleanest non-overlapping channel (1, 6, or 11).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// WiFi Channel Scanner (Analyze channel utilization)
#include <WiFi.h>

void analyzeChannels() {
  Serial.println("Scanning RF channels...");
  
  // 1. Scan for networks
  int numNetworks = WiFi.scanNetworks(false, false, false);
  
  if (numNetworks < 0) {
    Serial.println("Scan failed!");
    return;
  }
  
  // 2. Initialize channel counter array (index 1 to 13)
  int channelMap[14] = {0}; 
  
  // Count networks on each channel
  for (int i = 0; i < numNetworks; i++) {
    int chan = WiFi.channel(i);
    if (chan >= 1 && chan <= 13) {
      channelMap[chan]++;
    }
  }
  
  // 3. Print Channel Utilization Histogram
  Serial.println("\n--- WiFi Channel Utilization Chart ---");
  for (int chan = 1; chan <= 13; chan++) {
    Serial.printf("Channel %02d: ", chan);
    
    // Print stars representing number of networks
    int count = channelMap[chan];
    for (int star = 0; star < count; star++) {
      Serial.print("*");
    }
    
    // Fill space for formatting
    for (int space = count; space < 12; space++) {
      Serial.print(" ");
    }
    
    Serial.printf("(%d APs)\n", count);
  }
  Serial.println("---------------------------------------");
  
  // 4. Recommend the cleanest channel among non-overlapping channels (1, 6, 11)
  int c1 = channelMap[1];
  int c6 = channelMap[6];
  int c11 = channelMap[11];
  
  int bestChannel = 1;
  int minCount = c1;
  
  if (c6 < minCount) {
    minCount = c6;
    bestChannel = 6;
  }
  if (c11 < minCount) {
    minCount = c11;
    bestChannel = 11;
  }
  
  Serial.print("Recommendation -> Cleanest Non-Overlapping Channel: ");
  Serial.print(bestChannel);
  Serial.print(" (with ");
  Serial.print(minCount);
  Serial.println(" active networks)");
  Serial.println("=======================================\n");
  
  WiFi.scanDelete(); // Release memory
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\nWiFi channel analyzer active.");
  WiFi.mode(WIFI_STA);
  WiFi.disconnect();
  delay(100);
}

void loop() {
  analyzeChannels();
  
  // Repeat analysis every 10 seconds
  delay(10000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the channel usage histogram and recommendations print.

## Expected Output
Serial Monitor:
```
WiFi channel analyzer active.
Scanning RF channels...

--- WiFi Channel Utilization Chart ---
Channel 01: ***         (3 APs)
Channel 02:             (0 APs)
Channel 03:             (0 APs)
Channel 04:             (0 APs)
Channel 05:             (0 APs)
Channel 06: *           (1 APs)
Channel 07:             (0 APs)
Channel 08:             (0 APs)
Channel 09:             (0 APs)
Channel 10:             (0 APs)
Channel 11: **          (2 APs)
Channel 12:             (0 APs)
Channel 13:             (0 APs)
---------------------------------------
Recommendation -> Cleanest Non-Overlapping Channel: 6 (with 1 active networks)
=======================================
```

## Expected Canvas Behavior
* The Serial Monitor logs the channel usage histogram and cleanest channel recommendation every 10 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `channelMap[chan]++` | Increments the network counter for the corresponding channel. |
| `c1`, `c6`, `c11` | Reads the network counts for channels 1, 6, and 11. |
| `minCount` | Identifies the channel with the lowest count to recommend. |

## Hardware & Safety Concept: WiFi Co-channel Congestion
In the 2.4 GHz band, adjacent channels overlap and interfere with each other (e.g. channel 2 overlaps channels 1, 3, 4, 5, and 6). To avoid interference, network installations use the non-overlapping channels: **1, 6, and 11**. Analyzing channel usage helps select the optimal channel for a new network deployment.

## Try This! (Challenges)
1. **Interactive OLED Channel graph**: Add an OLED screen (Project 60) and display a bar graph showing the channel usage.
2. **Channel Congestion Alarm**: Sound a buzzer (GPIO 15) if all three non-overlapping channels are congested (> 5 APs each).
3. **RSSI-weighted analysis**: Weight the channel counts by signal strength, so closer networks contribute more to the congestion score.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The scanner does not detect any networks | Antenna shielded | Verify that the ESP32 is in range of an active WiFi router |
| Scan results are inconsistent | Transient interference | Average the results of three consecutive scans to filter out transient readings |
| The ESP32 crashes on loop | Memory leak | Ensure `WiFi.scanDelete()` is called after each scan to release memory |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [06 - ESP32 WiFi Network Scanner](06-esp32-wifi-network-scanner.md)
- [04 - ESP32 WiFi RSSI Signal strength logging](04-esp32-wifi-rssi-signal-strength-logging.md)
- [15 - ESP32 WiFi Signal Alert buzzer](15-esp32-wifi-signal-alert-buzzer.md)
