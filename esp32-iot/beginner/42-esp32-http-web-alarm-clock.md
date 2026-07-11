# 42 - ESP32 HTTP Web Alarm Clock

Build an NTP-synchronized digital alarm clock on the ESP32 that connects to a local WiFi network, syncs its clock with network time servers, displays the time on an I2C LCD, and triggers a buzzer alarm at a preset target hour and minute, using a button to silence the alarm.

## Goal
Learn how to parse local hours and minutes from RTC time structures, implement conditional alarm triggers, integrate buttons to clear active alarms, and build network-synced clocks.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A 16x2 I2C LCD is on I2C (GPIO 21/22). A buzzer is on GPIO 4, and an alarm-reset button on GPIO 12. The ESP32 syncs with an NTP server. When the local time matches the preset alarm time (e.g. 07:30), the buzzer sounds, and the LCD displays `ALARM ACTIVE!`. Pressing the button silences the alarm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Active Buzzer | VCC (+) | GPIO4 | Blue | Alarm horn |
| Push Button | Pin 1 / Pin 2 | GPIO12 / 3V3 | Orange / Red | Alarm Reset (active-HIGH) |
| Push Button | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the button with an external 10 kΩ pull-down resistor to GPIO 12. Power the LCD from the 5V Vin rail.

## Code
```cpp
// HTTP Web Alarm Clock (NTP synced + Alarm Trigger + Reset Button)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <WiFi.h>
#include "time.h"

LiquidCrystal_I2C lcd(0x27, 16, 2);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int BUZZER_PIN = 4;
const int BUTTON_PIN = 12;

// NTP Configuration (IST UTC+5:30 = 19800 seconds)
const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 19800; 
const int   daylightOffset_sec = 0;

// Preset Alarm Time Configuration (e.g. 07:30)
int alarmHour = 7;
int alarmMinute = 30;

// Alarm State Variables
bool alarmTriggered = false;
bool alarmSilenced = false;

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT); // External pull-down wired
  
  digitalWrite(BUZZER_PIN, LOW);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Web Alarm Clock");
  lcd.setCursor(0, 1);
  lcd.print("Connecting WiFi");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    lcd.print(".");
  }
  
  lcd.clear();
  lcd.print("Syncing Time...");
  
  // Initialize NTP time synchronization
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  
  struct tm timeinfo;
  while (!getLocalTime(&timeinfo)) {
    delay(500);
    lcd.print(".");
  }
  
  lcd.clear();
  lcd.print("Clock Synced!");
  delay(1500);
  lcd.clear();
}

void loop() {
  struct tm timeinfo;
  bool timeValid = getLocalTime(&timeinfo);
  
  // Read reset button
  bool buttonState = (digitalRead(BUTTON_PIN) == HIGH);
  
  if (timeValid) {
    // 1. Evaluate Alarm Trigger condition
    // Trigger alarm if current hour and minute match target
    if (timeinfo.tm_hour == alarmHour && timeinfo.tm_min == alarmMinute) {
      if (!alarmSilenced) {
        alarmTriggered = true;
      }
    } else {
      // Reset alarm state once the target minute passes
      alarmTriggered = false;
      alarmSilenced = false;
    }
    
    // 2. Evaluate Alarm Silence Button
    if (buttonState && alarmTriggered) {
      Serial.println("Alarm silenced by user.");
      alarmTriggered = false;
      alarmSilenced = true;
      digitalWrite(BUZZER_PIN, LOW);
      lcd.clear();
    }
    
    // 3. Actuate Alarm Siren (Pulsing sound: 500 ms ON, 500 ms OFF)
    if (alarmTriggered) {
      static unsigned long lastPulse = 0;
      static bool pulseState = LOW;
      unsigned long now = millis();
      
      if (now - lastPulse >= 500) {
        pulseState = !pulseState;
        digitalWrite(BUZZER_PIN, pulseState);
        lastPulse = now;
      }
      
      lcd.setCursor(0, 0);
      lcd.print("  ALARM ACTIVE! ");
      lcd.setCursor(0, 1);
      lcd.print("Press Button to C");
    } else {
      // Normal display mode
      digitalWrite(BUZZER_PIN, LOW);
      
      // Format time string (Row 0: e.g. "Time: 14:35:10")
      char timeStr[16];
      strftime(timeStr, sizeof(timeStr), "Time: %H:%M:%S", &timeinfo);
      lcd.setCursor(0, 0);
      lcd.print(timeStr);
      
      // Display alarm setting (Row 1: e.g. "Alarm: 07:30")
      lcd.setCursor(0, 1);
      char alarmStr[16];
      sprintf(alarmStr, "Alarm: %02d:%02d", alarmHour, alarmMinute);
      lcd.print(alarmStr);
    }
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **16x2 I2C LCD**, **Buzzer**, and **Button** onto the canvas.
2. Wire LCD to **GPIO21/GPIO22**, Buzzer to **GPIO4**, and Button to **GPIO12**.
3. Paste the code and click **Run**.
4. In simulation, set the alarm hour and minute to match the current UTC time (adjusted for offset). Watch the buzzer alarm sound and the LCD display `ALARM ACTIVE!`.
5. Click the button widget. The alarm stops.

## Expected Output
LCD Display:
```
Time: 07:30:00
Alarm: 07:30
```

LCD Display (Alarm Active):
```
  ALARM ACTIVE! 
Press Button to C
```

## Expected Canvas Behavior
* At the alarm time, the buzzer widget pulses green, and the LCD displays `ALARM ACTIVE!`.
* Clicking the button widget stops the buzzer.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `timeinfo.tm_hour == alarmHour` | Evaluates if the current hour matches the alarm target setting. |
| `alarmSilenced = true` | Latches the silenced flag to prevent the alarm from re-triggering during the same minute. |
| `now - lastPulse >= 500` | Controls the 1 Hz pulse rate (500 ms intervals) for the buzzer alarm. |

## Hardware & Safety Concept: Alarm Latch States
Alarm clocks must manage latch states. If the system only checked if the current hour and minute matched the target, the alarm would trigger repeatedly during that entire minute, even after the user silenced it. Implementing an **alarm silenced flag** (latch) prevents the alarm from re-triggering until the target minute passes.

## Try This! (Challenges)
1. **OLED Digital clock**: Upgrade the screen to an SSD1306 OLED (Project 36) and render a digital clock with a custom alarm icon.
2. **Snooze function**: Program the reset button to act as a 5-minute snooze button when clicked, and turn off the alarm completely when held for 3 seconds.
3. **RSSI Signal alarm**: Sound a warning beep if the WiFi connection drops.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound | Buzzer pin misconfigured | Confirm that the active buzzer is connected to GPIO 4 |
| Alarm triggers repeatedly during the same minute | Silenced flag missing | Verify that `alarmSilenced = true` is set when the reset button is clicked |
| Time is offset by 1 hour | DST setting incorrect | Verify that the DST offset parameter matches the local season |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [34 - ESP32 NTP Time Server sync](34-esp32-ntp-time-server-sync.md)
- [37 - ESP32 LCD Network Clock](37-esp32-lcd-network-clock.md)
- [15 - ESP32 WiFi Signal Alert buzzer](15-esp32-wifi-signal-alert-buzzer.md)
