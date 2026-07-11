# 50 - ESP32 Sound Sensor Wireless Clap Alert

Configure the ESP32 to connect to a local WiFi network, monitor a digital sound sensor to detect a clap event, sound a local alert buzzer, and transmit an HTTP POST alert packet to a remote server.

## Goal
Learn how to identify sound pulse transitions, manage transmit cooldown filters, configure HTTP clients, and transmit alerts.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A digital sound sensor is on GPIO 12, and a buzzer on GPIO 15. When a clap is detected (causing the sensor digital pin to pulse LOW), the buzzer sounds, and the ESP32 transmits an HTTP POST request containing `alert=CLAP_DETECTED` to `http://httpbin.org/post`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Sound Sensor Module | `sound_sensor` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Sound Sensor | OUT (Digital) | GPIO12 | Yellow | Sound detection trigger |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Status chime |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Standard digital sound sensors pulse LOW when sound is detected. Connect the digital output directly to GPIO 12.

## Code
```cpp
// Sound Sensor wireless clap alert (Clap detection uploader)
#include <WiFi.h>
#include <HTTPClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int SOUND_PIN = 12;
const int BUZZER_PIN = 15;

// Target URL API endpoint
const char* apiURL = "http://httpbin.org/post";

// Cooldown variables to prevent double-triggering on echo
unsigned long lastTriggerTime = 0;
const unsigned long TRIGGER_COOLDOWN_MS = 1500; // 1.5-second cooldown

void uploadAlert() {
  HTTPClient http;
  
  // Format payload
  String postData = "alert=CLAP_DETECTED&uptime_s=" + String(millis() / 1000);
  
  Serial.print("[HTTP] Connecting to: ");
  Serial.println(apiURL);
  
  if (!http.begin(apiURL)) {
    Serial.println("[HTTP] Connection initialization failed!");
    return;
  }
  
  http.addHeader("Content-Type", "application/x-www-form-urlencoded");
  
  Serial.print("[HTTP] Transmitting alert payload: ");
  Serial.println(postData);
  
  // Transmit POST request
  int httpResponseCode = http.POST(postData);
  
  if (httpResponseCode > 0) {
    Serial.print("[HTTP] Response Status Code: ");
    Serial.println(httpResponseCode);
    
    String payload = http.getString();
    Serial.println("[HTTP] --- Response Body ---");
    Serial.println(payload);
    Serial.println("[HTTP] ---------------------");
  } else {
    Serial.print("[HTTP] POST failed! Error: ");
    Serial.println(http.errorToString(httpResponseCode).c_str());
  }
  
  http.end();
  Serial.println("[HTTP] Connection closed.\n");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(SOUND_PIN, INPUT); // Sound sensor active-LOW
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Sound Alert System Starting");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  Serial.println("Sound sensor active. Waiting for clap...");
  Serial.println("==================================\n");
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    // 1. Read sound sensor digital state
    // Standard modules pulse LOW when sound is detected
    bool soundDetected = (digitalRead(SOUND_PIN) == LOW);
    unsigned long now = millis();
    
    // 2. Evaluate trigger with cooldown
    if (soundDetected) {
      if (now - lastTriggerTime >= TRIGGER_COOLDOWN_MS) {
        Serial.println("[Sound Alert] Clap detected! Triggering alert...");
        
        // Sound local alert beep
        digitalWrite(BUZZER_PIN, HIGH);
        delay(100);
        digitalWrite(BUZZER_PIN, LOW);
        
        // 3. Upload alert via HTTP POST
        uploadAlert();
        
        lastTriggerTime = now;
      }
    }
  } else {
    Serial.println("WiFi offline!");
    delay(1000);
  }
  
  delay(10); // Yield to system
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Sound Sensor**, and **Buzzer** onto the canvas.
2. Wire Sound Sensor to **GPIO12** and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Click the Sound Sensor widget to simulate a sound detection trigger. Watch the buzzer beep.
5. Inspect the Serial Monitor. Verify that the HTTP POST alert transmits.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Sound Alert System Starting
==================================
WiFi Connected successfully.
Sound sensor active. Waiting for clap...
==================================

[Sound Alert] Clap detected! Triggering alert...
[HTTP] Connecting to: http://httpbin.org/post
[HTTP] Transmitting alert payload: alert=CLAP_DETECTED&uptime_s=12
[HTTP] Response Status Code: 200
[HTTP] --- Response Body ---
{
  "form": {
    "alert": "CLAP_DETECTED", 
    "uptime_s": "12"
  }, 
  "headers": {
    "Content-Type": "application/x-www-form-urlencoded", 
    "Host": "httpbin.org"
  }, 
  "url": "http://httpbin.org/post"
}
[HTTP] ---------------------
[HTTP] Connection closed.
```

## Expected Canvas Behavior
* Actuating the simulated sound sensor pulses the buzzer widget green.
* The Serial Monitor logs the clap events and HTTP POST alerts.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `digitalRead(SOUND_PIN) == LOW` | Checks if the digital pin has pulsed LOW, indicating sound detection. |
| `now - lastTriggerTime >= 1500` | Enforces a 1.5-second cooldown to prevent double-triggering on echos. |
| `http.POST(postData)` | Transmits the alert payload to the server via an HTTP POST request. |

## Hardware & Safety Concept: Sound Sensor Calibration and Cooldowns
Sound sensors measure sound waves, which consist of alternating pressure peaks. A single clap generates a sound wave that oscillates for up to 500 ms. If a microcontroller reads the pin in a simple loop without a cooldown, it will interpret these oscillations as multiple distinct claps. Implementing a **cooldown timer** (e.g. 1.5 seconds) filters out these oscillations, ensuring a single trigger event.

## Try This! (Challenges)
1. **Double-Clap Switch**: Write code to toggle a relay (Project 46) only if two claps are detected within 1 second.
2. **OLED Log Dashboard**: Add an OLED screen (Project 60) and display a count of detected claps.
3. **Clap sound meter**: Connect the analog output of the sound sensor to GPIO 34 and log the raw sound volume.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The alert triggers continuously | Sensitivity threshold too low | Adjust the potentiometer screw on the physical sound sensor module to adjust sensitivity |
| Buzzer does not sound | Buzzer pin misconfigured | Confirm that the active buzzer is connected to GPIO 15 |
| Server rejects POST request (code 400 or 415) | Content-Type header missing | Verify that the Content-Type header is set correctly |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [32 - ESP32 HTTP POST Request](32-esp32-http-post-request.md)
- [44 - ESP32 Web-based LED toggle](44-esp32-web-based-led-toggle.md)
- [45 - ESP32 Web-based Buzzer chime trigger](45-esp32-web-based-buzzer-chime-trigger.md)
