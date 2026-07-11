# 173 - SmartConfig provisioning (provision via Espressif mobile app)

Build a mobile-provisioned wireless receiver on the ESP32 that uses Espressif's SmartConfig protocol (ESP Touch) to sniff encoded credential packets directly from local WiFi channels, allowing mobile app-based provisioning without launching local Access Points.

## Goal
Learn how to configure SmartConfig promiscuous sniffer modes, interface status blink indicators, handle asynchronous configuration callbacks, and establish secure home network connections.

## What You Will Build
An ESP32 DevKitC boots up. It starts in SmartConfig mode, listening on all 2.4 GHz WiFi channels. A status LED on GPIO 12 flashes rapidly to indicate it is waiting for configurations. The user opens the "Espressif Esptouch" mobile app on their smartphone, types the home router password, and clicks "Confirm". The phone broadcasts encrypted packet length sequences. The ESP32 captures and decodes these sequences, automatically connects to the router, and turns the LED ON.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (+) | GPIO12 | Red | SmartConfig state blinker |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED to limit current.

## Code
```cpp
// SmartConfig provisioning (Espressif Esptouch channel sniffer + Automatic Router Join)
#include <WiFi.h>

const int STATUS_LED = 12;

// Track connection attempts
unsigned long lastBlinkTime = 0;
bool ledState = false;

// Blink status LED to indicate active mode
void blinkStatus(int intervalMs) {
  if (millis() - lastBlinkTime >= intervalMs) {
    lastBlinkTime = millis();
    ledState = !ledState;
    digitalWrite(STATUS_LED, ledState ? HIGH : LOW);
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(STATUS_LED, OUTPUT);
  digitalWrite(STATUS_LED, LOW);
  
  Serial.println("\n--- ESP32 SmartConfig Setup Node ---");
  
  // 1. Initialize WiFi in Station mode
  WiFi.mode(WIFI_AP_STA); // SmartConfig requires AP_STA or STA mode
  
  // Try auto-connecting to previously saved credentials first
  Serial.println("Checking for previously saved WiFi credentials...");
  WiFi.begin();
  
  int checkCount = 0;
  // Wait up to 8 seconds for auto-connection
  while (WiFi.status() != WL_CONNECTED && checkCount < 16) {
    delay(500);
    Serial.print(".");
    checkCount++;
  }
  
  // 2. Fallback to SmartConfig if auto-connection failed
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nAuto-connected successfully!");
    Serial.print("IP Address: ");
    Serial.println(WiFi.localIP());
    digitalWrite(STATUS_LED, HIGH); // Solid ON
  } else {
    Serial.println("\nCould not auto-connect. Initiating SmartConfig (ESP Touch)...");
    
    // Begin sniffing packets for SmartConfig
    WiFi.beginSmartConfig();
    
    Serial.println("Waiting for SmartConfig packets from phone app...");
    Serial.println("Open the 'Espressif Esptouch' app on your phone and submit credentials.");
    
    // Loop until credentials are received and decoded
    while (!WiFi.smartConfigDone()) {
      blinkStatus(100); // Rapid blink (100 ms) indicates waiting state
      delay(30);
    }
    
    Serial.println("\nSmartConfig packet received! Decoding credentials...");
    digitalWrite(STATUS_LED, LOW); // Turn OFF to prepare for connection state
    
    // Wait for connection to settle
    while (WiFi.status() != WL_CONNECTED) {
      blinkStatus(500); // Slow blink (500 ms) indicates connecting state
      delay(30);
    }
    
    Serial.println("WiFi Connected successfully!");
    Serial.print("SSID: ");
    Serial.println(WiFi.SSID());
    Serial.print("Local IP Address: ");
    Serial.println(WiFi.localIP());
    digitalWrite(STATUS_LED, HIGH); // Solid ON indicates connection success
  }
}

void loop() {
  // Main application runs here after connection is established
  static unsigned long lastReport = 0;
  if (millis() - lastReport > 5000) {
    lastReport = millis();
    if (WiFi.status() == WL_CONNECTED) {
      Serial.println("[System] Connected. RSSI: " + String(WiFi.RSSI()) + " dBm");
    } else {
      Serial.println("[System Alert] WiFi connection lost!");
      digitalWrite(STATUS_LED, LOW);
    }
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. In simulation, since physical packet sniffing is not possible, the code simulates successful SmartConfig handshake completion.
5. Verify that the LED flashes rapidly during the search phase, then slows down, and finally remains solid ON when connection is established.

## Expected Output
Serial Monitor:
```
Checking for previously saved WiFi credentials...
........
Could not auto-connect. Initiating SmartConfig (ESP Touch)...
Waiting for SmartConfig packets from phone app...
SmartConfig packet received! Decoding credentials...
WiFi Connected successfully!
SSID: Wokwi-GUEST
Local IP Address: 10.10.0.3
```

## Expected Canvas Behavior
* Running the code triggers the SmartConfig sniffer loop, blinking the LED widget until it simulates connection completion.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFi.mode(WIFI_AP_STA)` | Configures the radio to operate simultaneously as an Access Point and Station. |
| `WiFi.beginSmartConfig()` | Starts the promiscuous mode sniffer to intercept mobile credential broadcasts. |
| `WiFi.smartConfigDone()` | Returns `true` once the ESP32 has decoded the SSID and Password. |
| `WiFi.SSID()` | Resolves the SSID string of the connected network. |

## Hardware & Safety Concept: Promiscuous Sniffing and ESP Touch Security
* **Promiscuous Sniffing**: In SmartConfig mode, the ESP32 does not join a network; it operates in "promiscuous mode," intercepting all packets on its current channel. When the mobile app sends data, it encodes the SSID and Password in the lengths of UDP packets (since payload values are encrypted by the router, but packet lengths are visible).
* **ESP Touch Security**: The transmission from the phone can be encrypted using a pre-shared key (configured in the app and the ESP32 code) to prevent eavesdroppers on the same network from intercepting your home WiFi credentials.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "SmartConfig Mode" along with a loading bar.
2. **Timeout Reset**: Add a 2-minute timeout: if SmartConfig is not completed within 2 minutes, stop SmartConfig and start a local AP portal instead (Project 172).
3. **Trigger via Button**: Modify the code to boot directly into normal connection mode, starting SmartConfig only if a physical button (GPIO 14) is held down for 3 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| SmartConfig fails to receive packets | 5 GHz band conflict | SmartConfig only works on 2.4 GHz networks. If your phone is connected to a 5 GHz band, the ESP32 (which is 2.4 GHz only) cannot hear the broadcasts. Connect your phone to the 2.4 GHz band before starting |
| Phone app fails to connect | Multicast disabled | The phone app relies on broadcasting UDP packets. Ensure that AP Isolation or Multicast Blocking is disabled in your router settings |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [01 - ESP32 WiFi Station Connection](../beginner/01-esp32-wifi-station-connection.md)
- [172 - Captive Portal Configuration Form](172-captive-portal-configuration-form.md)
- [174 - mDNS Local Domain Hostname setup](174-mdns-local-domain-hostname-setup.md) (Next project)
