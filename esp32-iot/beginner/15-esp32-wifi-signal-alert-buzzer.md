# 15 - ESP32 WiFi Signal Alert Buzzer

Build a wireless signal loss alarm on the ESP32 that monitors network connection strength (RSSI) in real time and sounds distinct buzzer warning sequences if the signal drops below a safety threshold or disconnects.

## Goal
Learn how to create warning beep sequences, parse signal levels, and implement network connection alarms.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An active buzzer is on GPIO 4. The ESP32 polls the RSSI value. If the signal drops below -80 dBm (weak signal), the buzzer sounds three rapid warning beeps. If the connection drops completely, the buzzer sounds a slow, repeating alarm beep.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Active Buzzer | VCC (+) | GPIO4 | Blue | Signal alert alarm |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the active buzzer directly to GPIO 4.

## Code
```cpp
// WiFi Signal Alert buzzer (Auditory signal drop alarm)
#include <WiFi.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int BUZZER_PIN = 4;
const int RSSI_CRITICAL_DBM = -80; // Safety threshold limit

unsigned long lastCheckTime = 0;
const unsigned long CHECK_INTERVAL_MS = 3000; // Check RSSI every 3 seconds

void beepSequence(int count, int durationMs, int gapMs) {
  for (int i = 0; i < count; i++) {
    digitalWrite(BUZZER_PIN, HIGH);
    delay(durationMs);
    digitalWrite(BUZZER_PIN, LOW);
    if (i < count - 1) {
      delay(gapMs);
    }
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\nWiFi Signal Monitor Active...");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  // Fast beep on setup start
  beepSequence(1, 100, 0);
}

void loop() {
  unsigned long now = millis();
  
  if (now - lastCheckTime >= CHECK_INTERVAL_MS) {
    wl_status_t status = WiFi.status();
    
    // 1. Evaluate Connection State
    if (status == WL_CONNECTED) {
      // Read signal strength
      int rssi = WiFi.RSSI();
      
      Serial.print("WiFi Connected | RSSI: ");
      Serial.print(rssi);
      Serial.println(" dBm");
      
      // 2. Evaluate Weak Signal Warning
      if (rssi < RSSI_CRITICAL_DBM) {
        Serial.println("Warning: Low Signal Strength! Sounding alert...");
        // 3 fast warning beeps
        beepSequence(3, 80, 80); 
      }
    } 
    else {
      // 3. Evaluate Disconnected Alarm
      Serial.println("Warning: WiFi Disconnected! Sounding alarm...");
      // 1 slow alarm beep
      beepSequence(1, 400, 0); 
    }
    
    lastCheckTime = now;
  }
  
  delay(20);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Buzzer** onto the canvas.
2. Wire Buzzer to **GPIO4**.
3. Paste the code and click **Run**.
4. Adjust the simulated WiFi network settings to lower the signal strength. Watch the buzzer sound three rapid beeps.
5. Turn the simulated network off. Watch the buzzer beep slowly.

## Expected Output
Serial Monitor:
```
WiFi Signal Monitor Active...
WiFi Connected | RSSI: -55 dBm
WiFi Connected | RSSI: -84 dBm
Warning: Low Signal Strength! Sounding alert...
Warning: WiFi Disconnected! Sounding alarm...
```

## Expected Canvas Behavior
* At boot, the buzzer widget pulses once.
* If the simulated signal strength drops below -80 dBm, the buzzer widget pulses green rapidly.
* If the simulated network drops, the buzzer widget pulses green slowly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFi.RSSI()` | Reads the active connection strength in dBm. |
| `rssi < RSSI_CRITICAL_DBM` | Triggers a warning if the signal drops below -80 dBm. |
| `beepSequence(...)` | Sounds distinct beep patterns (fast beeps for low signal, slow beeps for disconnected). |

## Hardware & Safety Concept: WiFi Signal Audits in Critical Nodes
Autonomous machines (such as remote rovers or security cameras) rely on WiFi connections to transmit coordinates or receive commands. If the vehicle drives into a wireless dead zone (where signal strength drops below -80 dBm), it can lose connection and become unresponsive. Monitoring the RSSI allows triggering warning beeps to alert the operator before the connection drops completely.

## Try This! (Challenges)
1. **Interactive OLED Dashboard**: Add an OLED screen (Project 60) and display the RSSI value and active alarm state.
2. **Dynamic Alert Threshold**: Add a potentiometer (Project 04) to allow manually setting the critical RSSI threshold.
3. **Signal Quality indicators**: Combine an LED (Project 14) to blink in sync with the buzzer alert sequence.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound | Buzzer pin misconfigured | Confirm that the active buzzer is connected to GPIO 4 |
| Alarm triggers continuously when close | Declination or configuration issue | Verify that the RSSI value is indeed below the threshold before sounding the alarm |
| Beep patterns are uneven | CPU load or timing delay | Keep the warning sequence non-blocking or limit execution delays |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [04 - ESP32 WiFi RSSI Signal strength logging](04-esp32-wifi-rssi-signal-strength-logging.md)
- [05 - ESP32 WiFi Disconnect and Auto-reconnect routine](05-esp32-wifi-disconnect-and-auto-reconnect-routine.md)
- [14 - ESP32 Wireless indicator LED](14-esp32-wireless-indicator-led.md)
