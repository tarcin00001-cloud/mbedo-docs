# 09 - ESP32 Access Point with Custom Security/Password

Configure the ESP32 in Access Point (AP) mode to host a secured local wireless network using WPA2-PSK encryption, limit maximum connections, and sound a buzzer beep when a client connects.

## Goal
Learn how to host a secured Access Point, enforce password length policies, limit client connection counts, and program connection alarms.

## What You Will Build
An ESP32 DevKitC hosts a secured local WiFi network named `ESP32-Secure-AP` with the password `SuperSecret123`. A buzzer is on GPIO 4. The network is configured to use channel 6 and limit connections to 4 clients. The buzzer beeps twice when a client connects.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Active Buzzer | VCC (+) | GPIO4 | Blue | Connection alert buzzer |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the active buzzer directly to GPIO 4.

## Code
```cpp
// Access Point with Custom Security/Password (WPA2-PSK Secured AP)
#include <WiFi.h>

const char* ap_ssid = "ESP32-Secure-AP";
const char* ap_password = "SuperSecret123"; // Must be at least 8 characters

const int BUZZER_PIN = 4;
const int AP_CHANNEL = 6;      // Use WiFi Channel 6
const int AP_HIDDEN = 0;       // 0 = Visible SSID, 1 = Hidden SSID
const int MAX_CLIENTS = 4;     // Limit connections to 4 clients

void playAlertTone() {
  digitalWrite(BUZZER_PIN, HIGH); delay(80);
  digitalWrite(BUZZER_PIN, LOW);  delay(50);
  digitalWrite(BUZZER_PIN, HIGH); delay(80);
  digitalWrite(BUZZER_PIN, LOW);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Secured Access Point Creator");
  Serial.println("==================================");
  
  // 1. Password length validation (WPA2 security requirement)
  if (strlen(ap_password) < 8) {
    Serial.println("Error: WPA2 password must be at least 8 characters long!");
    while(1) {}
  }
  
  // 2. Configure WiFi mode to Access Point
  WiFi.mode(WIFI_AP);
  
  // 3. Start the secured Access Point
  // softAP(ssid, password, channel, hidden, max_connection)
  bool success = WiFi.softAP(ap_ssid, ap_password, AP_CHANNEL, AP_HIDDEN, MAX_CLIENTS);
  
  if (success) {
    Serial.println("Secured Access Point is active!");
    Serial.print("SSID:       "); Serial.println(ap_ssid);
    Serial.print("Password:   "); Serial.println(ap_password);
    Serial.print("Channel:    "); Serial.println(AP_CHANNEL);
    Serial.print("Max Clients: "); Serial.println(MAX_CLIENTS);
    Serial.print("AP IP Address: "); Serial.println(WiFi.softAPIP());
  } else {
    Serial.println("Failed to start Secured Access Point!");
    while(1) {}
  }
  Serial.println("==================================\n");
}

void loop() {
  static int lastClientCount = 0;
  
  // 4. Monitor client connections
  int currentClientCount = WiFi.softAPgetStationNum();
  
  if (currentClientCount != lastClientCount) {
    Serial.print("[AP Alert] Connected clients changed. Count: ");
    Serial.println(currentClientCount);
    
    // Sound alert if a new client connects
    if (currentClientCount > lastClientCount) {
      Serial.println("[AP Alert] New client connected!");
      playAlertTone();
    } else {
      Serial.println("[AP Alert] Client disconnected.");
    }
    
    lastClientCount = currentClientCount;
  }
  
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Buzzer** onto the canvas.
2. Wire Buzzer to **GPIO4**.
3. Paste the code and click **Run**.
4. Verify that the AP IP, MAC, and security details print.
5. On hardware, connect to `ESP32-Secure-AP` using the password `SuperSecret123`. Watch the buzzer sound.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Secured Access Point Creator
==================================
Secured Access Point is active!
SSID:       ESP32-Secure-AP
Password:   SuperSecret123
Channel:    6
Max Clients: 4
AP IP Address: 192.168.4.1
==================================

[AP Alert] Connected clients changed. Count: 1
[AP Alert] New client connected!
```

## Expected Canvas Behavior
* The Serial Monitor logs the hosted AP configuration (SSID, IP, MAC) and tracks connected clients.
* If a simulated client connects, the buzzer widget pulses green.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `strlen(ap_password) < 8` | Validates password length; WPA2-PSK security requires at least 8 characters. |
| `WiFi.softAP(..., MAX_CLIENTS)` | Starts the secured AP, limiting connections to 4 clients to save RAM. |
| `currentClientCount > lastClientCount` | Triggers a buzzer alert when a new client connects. |

## Hardware & Safety Concept: Securing Access Point Configurations
Hosting an open Access Point allows anyone in range to connect to the ESP32. This exposes the system to security risks, such as Denial of Service (DoS) attacks. To protect the system:
1. **Encryption**: Always set a password of at least 8 characters to enable WPA2-PSK security.
2. **Limit Connections**: Limit the maximum number of connections to the minimum needed (e.g. 2 or 4) to protect memory resources.

## Try This! (Challenges)
1. **Interactive OLED Dashboard**: Add an OLED screen (Project 60) and display the SSID, password, and active client count.
2. **Dynamic Password reset**: Add a button on GPIO 12 that resets the AP password to a default string when pressed.
3. **Hidden secured SSID**: Configure `WiFi.softAP` parameters to hide the secured SSID broadcast.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| AP fails to start | Password too short | Confirm that the password is at least 8 characters long |
| Client cannot connect | Limit reached | Verify that the number of connected clients has not reached the limit |
| The network is not visible to devices | Channel overlap | Change the channel parameter to a cleaner channel (e.g. 1, 6, or 11) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [08 - ESP32 Access Point mode creation](08-esp32-access-point-mode-creation.md)
- [10 - ESP32 Access Point IP DHCP range logger](10-esp32-access-point-ip-dhcp-range-logger.md)
- [01 - ESP32 WiFi Station Connection](01-esp32-wifi-station-connection.md)
