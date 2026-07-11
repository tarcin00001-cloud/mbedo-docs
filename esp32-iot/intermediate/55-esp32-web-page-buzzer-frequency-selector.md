# 55 - Web Page Buzzer Frequency Selector

Build a web-controlled sound synthesizer on the ESP32 that hosts a control panel, parses target frequency parameters from HTTP requests (`/set?freq=1500`), and drives a passive buzzer using the standard `tone()` API.

## Goal
Learn how to parse integer query parameters, generate variable PWM frequencies using the `tone()` API, build interactive frequency selection forms, and handle client redirects.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. A passive buzzer is on GPIO 15. Navigating to the ESP32's IP address displays a webpage with buttons for preset frequencies (500 Hz, 1000 Hz, 2000 Hz, and Silent) and an input slider to select any frequency between 100 Hz and 5000 Hz. Changing the frequency plays that tone on the buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Passive Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Passive Buzzer | VCC (+) | GPIO15 | Blue | Audio signal channel |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the passive buzzer directly to GPIO 15.

## Code
```cpp
// Web Page Buzzer frequency selector (Web Server PWM Frequency Synthesizer)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int BUZZER_PIN = 15;

// Active frequency state variables
int activeFrequency = 0; // 0 represents Silent

WebServer server(80);

// Root URL Route Handler ("/")
void handleRoot() {
  // Compile HTML page with frequency selection options
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Buzzer Frequency Controller</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; margin-bottom: 20px; }\n";
  html += "  .status { font-size: 18px; margin: 20px 0; color: #94a3b8; }\n";
  html += "  .status-val { font-weight: bold; color: " + String(activeFrequency > 0 ? "#10b981" : "#ef4444") + "; }\n";
  html += "  .presets { display: flex; flex-wrap: wrap; justify-content: center; gap: 10px; margin-bottom: 25px; }\n";
  html += "  .btn { display: inline-block; padding: 10px 18px; font-size: 14px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; text-decoration: none; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "  .btn-silent { background-color: #ef4444; }\n";
  html += "  .btn-silent:hover { background-color: #dc2626; }\n";
  html += "  .slider-form { border-top: 1px solid #334155; padding-top: 25px; margin-top: 20px; }\n";
  html += "  .slider { width: 100%; margin: 15px 0; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Buzzer Tone Synthesizer</h1>\n";
  
  // Status display
  html += "  <p class=\"status\">Active Tone: <span class=\"status-val\">";
  if (activeFrequency > 0) {
    html += String(activeFrequency) + " Hz";
  } else {
    html += "SILENT";
  }
  html += "  </span></p>\n";
  
  // Preset buttons
  html += "  <div class=\"presets\">\n";
  html += "    <a href=\"/set?freq=500\" class=\"btn\">500 Hz</a>\n";
  html += "    <a href=\"/set?freq=1000\" class=\"btn\">1000 Hz</a>\n";
  html += "    <a href=\"/set?freq=2000\" class=\"btn\">2000 Hz</a>\n";
  html += "    <a href=\"/set?freq=0\" class=\"btn btn-silent\">SILENCE</a>\n";
  html += "  </div>\n";
  
  // Slider form for custom frequency
  html += "  <div class=\"slider-form\">\n";
  html += "    <form action=\"/set\" method=\"GET\">\n";
  html += "      <label for=\"freqRange\">Custom Frequency (100 - 5000 Hz):</label><br>\n";
  html += "      <input type=\"range\" id=\"freqRange\" name=\"freq\" min=\"100\" max=\"5000\" value=\"" + String(activeFrequency > 0 ? activeFrequency : 1000) + "\" class=\"slider\" oninput=\"updateSliderVal(this.value)\">\n";
  html += "      <span id=\"sliderVal\">" + String(activeFrequency > 0 ? activeFrequency : 1000) + " Hz</span><br><br>\n";
  html += "      <button type=\"submit\" class=\"btn\">Apply Custom Tone</button>\n";
  html += "    </form>\n";
  html += "  </div>\n";
  
  html += "</div>\n";
  
  // JS script to update text value dynamically as slider moves
  html += "<script>\n";
  html += "  function updateSliderVal(val) {\n";
  html += "    document.getElementById('sliderVal').innerText = val + ' Hz';\n";
  html += "  }\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// Frequency setting handler ("/set")
void handleSet() {
  // 1. Verify if query parameter "freq" is present
  if (server.hasArg("freq")) {
    int freq = server.arg("freq").toInt();
    
    // 2. Validate frequency range limits
    if (freq >= 100 && freq <= 5000) {
      activeFrequency = freq;
      
      // Play tone using core wrapper
      tone(BUZZER_PIN, activeFrequency);
      Serial.printf("[Server] Frequency applied: %d Hz\n", activeFrequency);
    } 
    else if (freq == 0) {
      activeFrequency = 0;
      
      // Stop tone using core wrapper
      noTone(BUZZER_PIN);
      Serial.println("[Server] Buzzer Silenced.");
    } 
    else {
      Serial.printf("[Server Error] Out of bounds frequency attempted: %d Hz\n", freq);
    }
  }
  
  // Redirect back to root page to update status
  server.sendHeader("Location", "/");
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // In ESP32 Arduino Core, tone() pin should be set as output
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\nESP32 Tone Web Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  server.on("/set", handleSet);
  
  server.begin();
  Serial.print("Web Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Buzzer** onto the canvas.
2. Wire Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Click the buttons or adjust the slider to control the buzzer pitch.

## Expected Output
Serial Monitor:
```
ESP32 Tone Web Server Starting...
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
[Server] Frequency applied: 1000 Hz
[Server] Frequency applied: 2000 Hz
[Server] Buzzer Silenced.
```

## Expected Canvas Behavior
* Selecting a frequency on the web page updates the tone sounded by the passive buzzer widget.
* Clicking Silence turns off the sound.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `tone(BUZZER_PIN, ...)` | Generates a square wave of the specified frequency on the buzzer pin. |
| `noTone(BUZZER_PIN)` | Stops wave generation, silencing the pin output. |
| `server.arg("freq").toInt()` | Parses the frequency query parameter string to an integer value. |

## Hardware & Safety Concept: Variable Audio Frequency Synthesis
Passive buzzers require an alternating current (AC) signal to flex their internal piezo element and generate sound waves. The frequency of the signal determines the pitch of the tone. Generating frequencies below 100 Hz yields crackling noises, while frequencies above 5000 Hz can exceed the transducer's limits or human hearing range, wasting power. Restricting parameters protects the hardware.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a simulated sound wave matching the frequency.
2. **Frequency Chime Pattern**: Add a button to play a startup chime (e.g. 3 rising tones).
3. **Emergency Auto-Timeout**: Add code to silence the buzzer automatically after 5 seconds of continuous playback to prevent coil overheating.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound | Buzzer pin misconfigured | Confirm that the passive buzzer is connected to GPIO 15 |
| Sound is soft or distorted | Resistor in series | Ensure the passive buzzer is powered directly from the output pin (verify maximum current limits) |
| Web page responds slowly | Blocking delays | Ensure the code does not block client handling inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [45 - ESP32 Web-based Buzzer chime trigger](../beginner/45-esp32-web-based-buzzer-chime-trigger.md)
- [54 - ESP32 Web Page Relay slider switch](54-esp32-web-page-relay-slider-switch.md)
- [56 - ESP32 Web Page Servo angle controller](56-esp32-web-page-servo-angle-controller-oled-display-feedback.md) (Next project)
