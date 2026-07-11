# 113 - DS3231 RTC Alarm System

Build a real-time scheduler and alarm node that reads time from a DS3231 RTC module and triggers an active buzzer on GPIO 14 when a configured alarm time is reached.

## Goal
Learn how to parse time structures, write comparison logic for scheduling events, and drive alarm buzzers based on precise RTC time values without using C++ loops.

## What You Will Build
A digital alarm clock system. The ARIES board continuously tracks time. When the clock hits 00 seconds of any minute, the system sounds a pulsing buzzer alarm. The alarm automatically silences after 5 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DS3231 Real Time Clock | `ds3231` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS3231 RTC | VCC | 3V3 | Red | RTC module power (3.3V) |
| DS3231 RTC | GND | GND | Black | Ground reference |
| DS3231 RTC | SDA | SDA0 (GP17) | Blue | I2C Data line |
| DS3231 RTC | SCL | SCL0 (GP16) | Yellow | I2C Clock line |
| Active Buzzer | VCC | GPIO 14 | Orange | Output high triggers alarm |
| Active Buzzer | GND | GND | Black | Ground reference |

> **Wiring tip:** The DS3231 RTC connects to the hardware I2C0 pins. Always share the ground connections. The active buzzer can be driven directly from GPIO 14.

## Code
```cpp
#include <Wire.h>
#include <RTClib.h>

RTC_DS3231 rtc;

const int BUZZER_PIN = 14;   // Alarm Buzzer on GPIO 14

// Alarm triggers at 00 seconds of every minute for simulation visibility
const int ALARM_SECOND = 0; 
int isAlarmActive = 0;

void setup() {
  Wire.begin();
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW); // Start with buzzer OFF

  rtc.begin();
  if (rtc.lostPower()) {
    rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
  }
}

void loop() {
  DateTime now = rtc.now();

  // Check if current time matches the alarm second
  if (now.second() == ALARM_SECOND) {
    isAlarmActive = 1;
  }

  // Turn off the alarm after 5 seconds
  if (now.second() >= 5 && now.second() < 55) {
    isAlarmActive = 0;
  }

  // Drive active buzzer alarm pattern
  if (isAlarmActive == 1) {
    // Pulse the buzzer depending on current second to create a beeping sound
    if (now.second() % 2 == 0) {
      digitalWrite(BUZZER_PIN, HIGH);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
    }
  } else {
    digitalWrite(BUZZER_PIN, LOW);
  }

  delay(200); // Polling interval
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DS3231 Real Time Clock**, and **Active Buzzer** components onto the canvas.
2. Wire the DS3231: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
3. Wire the Buzzer: **VCC** to **GPIO 14** and **GND** to **GND**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Time: 12:45:58 - Status: OK
Time: 12:45:59 - Status: OK
Time: 12:46:00 - ALARM TRIGGERED!
Time: 12:46:05 - ALARM SILENCED
```

## Expected Canvas Behavior
* The DS3231 counts seconds in the background.
* When the seconds counter reaches `:00`, the active buzzer starts sounding in a pulsing pattern.
* When the seconds counter reaches `:05`, the buzzer shuts off and remains silent until the next minute.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `rtc.begin()` | Initialized connection to the DS3231 RTC on I2C0. |
| `rtc.now()` | Queries the current year, month, day, hour, minute, and second. |
| `now.second() == ALARM_SECOND` | Compares current clock time to the scheduled trigger second. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives GPIO 14 high to sound the alarm. |

## Hardware & Safety Concept
* **Time-critical Triggers**: In software design, scheduling events requires matching time metrics (hours, minutes, and seconds). Always write bounds checks or latch flags rather than simple equality checks (e.g. `==`), to ensure that if a loop cycles slowly or missed a second, the alarm will still fire.
* **Backup Battery Maintenance**: DS3231 modules contain a battery charger circuit designed for rechargeable LIR2032 cells. If using non-rechargeable CR2032 batteries, physical safety dictates removing the module's charging resistor or diode to prevent the battery from swelling or leaking.

## Try This! (Challenges)
1. **Specific Alarm Time**: Change the code to trigger the alarm at a specific hour and minute (e.g., 08:30:00) instead of every minute.
2. **Alarm Disable Button**: Connect a push button on GPIO 16. If the alarm is sounding, pressing the button should silence the buzzer immediately.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound when second reaches 0 | Alarm flag cleared too early | Ensure loop delay is around 200 ms. If delay is too high, it might skip the `:00` second entirely. |
| RTC clock does not keep time when powered down | Missing backup battery | Install a CR2032 battery cell on the RTC module. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [70 - Active Buzzer Chime Scale Generator](70-active-buzzer-chime-scale-generator.md)
- [112 - DS3231 Real Time Clock Display](112-ds3231-real-time-clock-display.md)
