# 23 - ESP32 WiFi Sleep State Selector

Configure the ESP32 to modify its internal WiFi RF power-saving modes (Modem-Sleep states), selecting between maximum throughput and low-power options, and log the state transitions to the Serial Monitor.

## Goal
Learn how to configure ESP32 WiFi power-saving states (Modem-Sleep), balance RF power consumption, and evaluate latency trade-offs.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It logs the default WiFi power save state. It then changes the sleep configuration to Maximum Modem-Sleep (`WIFI_PS_MAX_MODEM`) to reduce power consumption, and logs the configuration changes.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// WiFi Sleep State selector (WiFi power save modes)
#include <WiFi.h>
#include "esp_wifi.h"

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Helper to translate WiFi power save enums to readable strings
const char* getPowerSaveModeString(wifi_ps_type_t type) {
  switch (type) {
    case WIFI_PS_NONE:       return "WIFI_PS_NONE (No power save. Continuous RF active. High current ~120mA)";
    case WIFI_PS_MIN_MODEM:  return "WIFI_PS_MIN_MODEM (Min Modem Sleep. Periodic radio shutdown. Balances latency ~40mA)";
    case WIFI_PS_MAX_MODEM:  return "WIFI_PS_MAX_MODEM (Max Modem Sleep. Extreme radio shutdown. Low power ~15mA, high latency)";
    default:                 return "UNKNOWN_SLEEP_MODE";
  }
}

void auditPowerSaveMode() {
  wifi_ps_type_t activeMode;
  
  // 1. Query the active WiFi power save mode from ESP-IDF registers
  esp_err_t err = esp_wifi_get_ps(&activeMode);
  
  if (err == ESP_OK) {
    Serial.print("[Power Audit] Active Mode: ");
    Serial.println(getPowerSaveModeString(activeMode));
  } else {
    Serial.println("[Power Audit] Failed to query power save mode!");
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 WiFi Sleep State Selector");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  // 2. Audit initial default power save mode
  auditPowerSaveMode();
  delay(1000);
  
  // 3. Configure Maximum Modem-Sleep mode
  // Reduces RF current consumption to a fraction of active mode
  Serial.println("\n[Action] Configuring WIFI_PS_MAX_MODEM sleep state...");
  esp_err_t err = esp_wifi_set_ps(WIFI_PS_MAX_MODEM);
  
  if (err == ESP_OK) {
    Serial.println("[Success] Max Modem-Sleep applied successfully.");
  } else {
    Serial.println("[Error] Failed to apply power save mode!");
  }
  
  // 4. Re-audit mode to confirm changes
  auditPowerSaveMode();
  Serial.println("==================================\n");
}

void loop() {
  // Main loop remains active. The CPU is fully powered, 
  // but the internal WiFi radio sleeps periodically in the background.
  delay(10000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the power save mode transition logs.

## Expected Output
Serial Monitor:
```
==================================
ESP32 WiFi Sleep State Selector
==================================
WiFi Connected successfully.
[Power Audit] Active Mode: WIFI_PS_MIN_MODEM (Min Modem Sleep. Periodic radio shutdown. Balances latency ~40mA)

[Action] Configuring WIFI_PS_MAX_MODEM sleep state...
[Success] Max Modem-Sleep applied successfully.
[Power Audit] Active Mode: WIFI_PS_MAX_MODEM (Max Modem Sleep. Extreme radio shutdown. Low power ~15mA, high latency)
==================================
```

## Expected Canvas Behavior
* The Serial Monitor logs the initial WiFi power save mode and the transition to the new mode on boot.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `esp_wifi_get_ps(...)` | Queries the current power-saving mode from the ESP32 wireless controller registers. |
| `esp_wifi_set_ps(...)` | Configures the target power-saving mode (e.g. `WIFI_PS_MAX_MODEM`). |
| `WIFI_PS_MAX_MODEM` | Activates maximum modem-sleep, which periodically shuts down the radio receiver during idle times. |

## Hardware & Safety Concept: WiFi Modem-Sleep vs Deep Sleep
The ESP32 WiFi radio consumes up to 240 mA when transmitting. If powered from a battery, this can exhaust the charge within hours. To extend battery life:
1. **Modem-Sleep**: Shuts down the WiFi radio periodically while keeping the CPU active. This allows running control code while reducing current draw.
2. **Deep Sleep**: Shuts down both the CPU and the radio (Project 177). Select the mode that matches your project's power and latency requirements.

## Try This! (Challenges)
1. **Interactive Mode Switcher**: Add a button on GPIO 12 to cycle through the three WiFi power save modes.
2. **OLED Status Display**: Add an OLED screen (Project 60) and display the active WiFi sleep state.
3. **Latency Diagnostic**: Ping a host (Project 22) before and after enabling max modem sleep to measure the latency difference.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `esp_wifi_set_ps` returns error | WiFi stack offline | Ensure the WiFi client is connected to a network before modifying power-saving modes |
| Max Modem-Sleep disconnects frequently | Router beacon interval too long | Switch to `WIFI_PS_MIN_MODEM` if the router drops connections during sleep |
| Ping latency is high in Max sleep | Expected behavior | Max Modem-Sleep trades latency for power savings; switch to `WIFI_PS_NONE` for high-throughput tasks |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [177 - ESP32 Deep Sleep Low-power Wakeup](../expert/177-esp32-deep-sleep-low-power-wakeup.md)
- [178 - ESP32 Light Sleep Low-power Wakeup](../expert/178-esp32-light-sleep-low-power-wakeup.md)
- [04 - ESP32 WiFi RSSI Signal strength logging](04-esp32-wifi-rssi-signal-strength-logging.md)
