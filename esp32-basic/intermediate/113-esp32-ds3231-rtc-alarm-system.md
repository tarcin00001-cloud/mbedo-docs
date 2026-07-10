# 113 - ESP32 DS3231 RTC Alarm System

Build a daily scheduled alarm system that monitors a DS3231 Real Time Clock and triggers an audio-visual warning buzzer and LED when a target time is reached.

## Goal
Learn how to perform time comparisons in code, configure daily schedule triggers, and actuate alarm outputs based on clock values.

## What You Will Build
A DS3231 RTC is connected to the I2C bus. A buzzer is connected to GPIO 15, and an LED to GPIO 5. The code checks the current time. When the seconds count matches a target alarm time (e.g. 30 seconds of every minute), the LED flashes and the buzzer beeps for 5 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DS3231 Real Time Clock Module | `rtc_ds3231` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS3231 Module | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| DS3231 Module | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| LED (Red) | Anode (+) | GPIO5 via 330 Ω | Orange | Alarm light indicator |
| LED (Red) | Cathode (−) | GND | Black | Ground reference |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm horn output |
| Active Buzzer | GND (−) | GND | Black | Ground reference |

> **Wiring tip:** The active buzzer and LED are connected to GPIO 15 and 5. The RTC connects to the standard ESP32 I2C pins.

## Code
```cpp
// DS3231 RTC Alarm System
#include <Wire.h>
#include <RTClib.h>

const int LED_PIN = 5;
const int BUZZER_PIN = 15;

RTC_DS3231 rtc;

// Target alarm schedule: triggers when seconds reach this value
// (In a physical clock, you would check specific hour and minute values)
const int ALARM_TRIGGER_SECOND = 30; 
const int ALARM_DURATION_S = 5;       // Alarm runs for 5 seconds

void setup() {
  Serial.begin(115200);
  
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(LED_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  
  if (!rtc.begin()) {
    Serial.println("RTC initialization failed!");
    while(1) {}
  }
  
  if (rtc.lostPower()) {
    rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
  }
  
  Serial.println("RTC Alarm System ready.");
  Serial.print("Alarm scheduled for second: "); Serial.println(ALARM_TRIGGER_SECOND);
}

void loop() {
  DateTime now = rtc.now();
  
  // Format print time to serial
  Serial.print("Current Time: ");
  Serial.print(now.hour()); Serial.print(":");
  Serial.print(now.minute()); Serial.print(":");
  Serial.println(now.second());
  
  // Evaluate if current time matches the scheduled window
  // Checks if the seconds falls inside [Trigger, Trigger + Duration]
  int elapsed = now.second() - ALARM_TRIGGER_SECOND;
  
  if (elapsed >= 0 && elapsed < ALARM_DURATION_S) {
    Serial.println("!!! ALARM ACTIVE !!!");
    
    // Pulse alarm outputs
    digitalWrite(LED_PIN, HIGH);
    digitalWrite(BUZZER_PIN, HIGH);
    delay(200);
    digitalWrite(LED_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);
    delay(200);
  } else {
    // Idle standby
    digitalWrite(LED_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);
    delay(1000); // Poll once per second
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DS3231 RTC**, **LED**, and **Buzzer** onto the canvas.
2. Wire I2C pins, LED to **GPIO5**, and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Observe the current seconds counting up. Watch the buzzer beep and LED flash when the seconds count hits 30.

## Expected Output
Serial Monitor:
```
RTC Alarm System ready.
Current Time: 12:05:28
Current Time: 12:05:29
Current Time: 12:05:30
!!! ALARM ACTIVE !!!
Current Time: 12:05:35
```

## Expected Canvas Behavior
* During standard counting, the LED and buzzer are OFF.
* When the RTC seconds clock reaches 30, the LED and buzzer widgets pulse for 5 seconds.
* The alarm shuts off automatically when the seconds count reaches 35.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `now.second()` | Retrieves the current seconds integer value from the RTC. |
| `elapsed >= 0 && elapsed < ALARM_DURATION_S` | Determines if the clock is currently inside the active alarm time window. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Sounds the active buzzer alarm. |

## Hardware & Safety Concept: Alarm Schedules and Time Window Matching
Checking for absolute equality (`now.second() == ALARM_TRIGGER_SECOND`) can fail if the microcontroller is executing a slow routine (e.g. querying a slow sensor) and misses that exact second. To prevent missed alarms, it is safer to evaluate a **time window** (`now.second() >= TRIGGER && now.second() < TRIGGER + DURATION`) so the alarm triggers even if the first second was missed.

## Try This! (Challenges)
1. **Daily Alarm Schedule**: Modify the code to trigger the alarm at a specific hour and minute (e.g. 07:30:00).
2. **Button Alarm Snooze**: Add a button on GPIO 4. If pressed during the alarm, silence the buzzer and delay the alarm for 10 seconds.
3. **Interactive LCD HUD**: Add an LCD display showing the current time and a Bell icon representing the alarm status.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers multiple times | Double looping | Ensure your check window handles time rollover (e.g. at 59s to 0s transitions) |
| Alarm misses the trigger second | Heavy delay blocks inside loop | Avoid long blocking delays inside the loop to prevent missing seconds changes |
| Time does not match local time | RTC not synchronized | Ensure `rtc.lostPower()` block is active to calibrate compiling timestamps |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [112 - ESP32 DS3231 Real Time Clock Display](112-esp32-ds3231-real-time-clock-display.md)
- [06 - ESP32 Active Buzzer Beep Alarm Pattern](../beginner/06-esp32-active-buzzer-beep-alarm-pattern.md)
- [103 - ESP32 Smart Doorbell Alert](103-esp32-smart-doorbell-alert.md)
