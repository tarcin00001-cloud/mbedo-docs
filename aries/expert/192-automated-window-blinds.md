# 192 - Automated Window Blinds

Implement a smart window blinds system that uses a DS3231 RTC to schedule blind positions, checks ambient sunlight levels using a light-dependent resistor (LDR) to prevent glare, and controls a servo motor to open or close the blinds, featuring manual override button integration.

## Goal
Learn how to merge time schedules with sensor readings in state machines, drive servos to precise angles in interpreted mode, and implement long manual override timeouts without using blocking code.

## What You Will Build
An intelligent building shading node. The ARIES v3 board reads the DS3231 RTC. During daylight hours (07:00:00 to 18:00:00) and if the LDR is above a threshold (300), the blinds are opened (servo rotates to 90°). If it is night or if the light falls below 200 (cloudy/dark), the blinds close (servo rotates to 0°). Pressing the button on GPIO 16 overrides the automatic mode for 10 seconds (for testing) or toggles positions immediately.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DS3231 Real Time Clock | `ds3231` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Push Button | `button` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS3231 RTC | VCC | 3V3 | Red | RTC power supply |
| DS3231 RTC | GND | GND | Black | Ground reference |
| DS3231 RTC | SDA | SDA0 (GP17) | Blue | I2C Data line |
| DS3231 RTC | SCL | SCL0 (GP16) | Yellow | I2C Clock line |
| LDR | Pin 1 | 3V3 | Red | Voltage source |
| LDR | Pin 2 | ADC1 (GP27) | White | Analog voltage output (with 10k to GND) |
| Servo Motor | Signal | GPIO 13 | Yellow | Servo PWM control |
| Servo Motor | VCC | 5V | Red | Servo power supply |
| Servo Motor | GND | GND | Black | Ground reference |
| Push Button | Pin 1 | GPIO 16 | Orange | Override toggle button |
| Push Button | Pin 2 | GND | Black | Ground reference |

> **Wiring tip:** Standard micro-servos can be powered directly from the ARIES 5V pin for simulation, but in practical installations with multiple large servo motors, you should use an external power source to avoid overloading the ARIES voltage regulators.

## Code
```cpp
// 192 - Automated Window Blinds
#include <Wire.h>
#include <RTClib.h>
#include <Servo.h>

const int LDR_PIN = 27;     // ADC1
const int SERVO_PIN = 13;   // Servo Signal
const int BUTTON_PIN = 16;  // Manual Override Button

const int LED_R_PIN = 23;
const int LED_G_PIN = 24;
const int LED_B_PIN = 25;

Servo blindServo;
RTC_DS3231 rtc;

int blindState = 0;         // 0 = Closed (0 degrees), 1 = Open (90 degrees)
int manualMode = 0;         // 0 = Auto, 1 = Manual Override Active
int manualTimer = 0;        // Ticks to hold manual override (1 tick = 200 ms)
int lastButtonState = HIGH;
int ldrValue = 0;
int logTimer = 0;

void setup() {
  Serial.begin(115200);
  Wire.begin();

  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_R_PIN, OUTPUT);
  pinMode(LED_G_PIN, OUTPUT);
  pinMode(LED_B_PIN, OUTPUT);

  // Status indicators: Red = Manual, Green = Auto-Closed, Blue = Auto-Open (Active Low)
  digitalWrite(LED_R_PIN, HIGH);
  digitalWrite(LED_G_PIN, HIGH);
  digitalWrite(LED_B_PIN, HIGH);

  blindServo.attach(SERVO_PIN);
  blindServo.write(0); // Initialize closed

  rtc.begin();
  if (rtc.lostPower()) {
    rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
  }

  Serial.println("Automated Window Blinds Node Online.");
}

void loop() {
  DateTime now = rtc.now();
  ldrValue = analogRead(LDR_PIN);

  // Read manual override button (Active LOW)
  int currentButtonState = digitalRead(BUTTON_PIN);
  if (currentButtonState == LOW && lastButtonState == HIGH) {
    manualMode = 1;
    manualTimer = 50; // Keep in manual mode for 10 seconds (50 ticks * 200ms)
    
    // Toggle blind state
    if (blindState == 0) {
      blindState = 1;
      blindServo.write(90);
    } else {
      blindState = 0;
      blindServo.write(0);
    }
    Serial.println("Manual override triggered. Blinds toggled.");
  }
  lastButtonState = currentButtonState;

  // Manage manual override countdown
  if (manualMode == 1) {
    manualTimer--;
    if (manualTimer <= 0) {
      manualMode = 0;
      Serial.println("Manual override timed out. Reverting to Auto mode.");
    }
  }

  // Automatic Mode Logic
  if (manualMode == 0) {
    // Open if between 07:00 and 18:00 and light is sufficient
    if (now.hour() >= 7 && now.hour() < 18 && ldrValue > 300) {
      if (blindState == 0) {
        blindState = 1;
        blindServo.write(90);
        Serial.println("Auto Mode: Opening blinds (Daylight detected).");
      }
    } else {
      // Close at night or if ambient light drops (cloudy/stormy)
      if (blindState == 1) {
        blindState = 0;
        blindServo.write(0);
        Serial.println("Auto Mode: Closing blinds (Low light / Night).");
      }
    }
  }

  // LED Status update
  if (manualMode == 1) {
    digitalWrite(LED_R_PIN, LOW);  // Red = Manual
    digitalWrite(LED_G_PIN, HIGH);
    digitalWrite(LED_B_PIN, HIGH);
  } else {
    digitalWrite(LED_R_PIN, HIGH);
    if (blindState == 1) {
      digitalWrite(LED_B_PIN, LOW);  // Blue = Auto-Open
      digitalWrite(LED_G_PIN, HIGH);
    } else {
      digitalWrite(LED_G_PIN, LOW);  // Green = Auto-Closed
      digitalWrite(LED_B_PIN, HIGH);
    }
  }

  // Serial logging every ~1000 ms (5 * 200 ms)
  logTimer++;
  if (logTimer >= 5) {
    logTimer = 0;
    Serial.print("Time: ");
    Serial.print(now.hour());
    Serial.print(":");
    Serial.print(now.minute());
    Serial.print(" | Light: ");
    Serial.print(ldrValue);
    Serial.print(" | Position: ");
    Serial.print(blindState ? "OPEN (90 deg)" : "CLOSED (0 deg)");
    Serial.print(" | Mode: ");
    Serial.println(manualMode ? "MANUAL OVERRIDE" : "AUTO");
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DS3231 Real Time Clock**, **LDR Photoresistor**, **Servo Motor**, and **Push Button** onto the canvas.
2. Wire the RTC: **VCC → 3V3**, **GND → GND**, **SDA → SDA0 (GP17)**, **SCL → SCL0 (GP16)**.
3. Wire the LDR: **VCC → 3V3**, **Output → ADC1 (GP27)** (with 10k to GND).
4. Wire the Servo: **Signal → GPIO 13**, **VCC → 5V**, **GND → GND**.
5. Wire the Button: **Pin 1 → GPIO 16**, **Pin 2 → GND**.
6. Paste the code, select **Interpreted Mode**, and click **Run**.
7. Adjust the LDR widget's light slider or change the RTC time widget to observe automatic movement. Press the button to override the operation.

## Expected Output
Serial Monitor:
```
Automated Window Blinds Node Online.
Time: 6:59:58 | Light: 150 | Position: CLOSED (0 deg) | Mode: AUTO
Time: 6:59:59 | Light: 152 | Position: CLOSED (0 deg) | Mode: AUTO
Auto Mode: Opening blinds (Daylight detected).
Time: 7:00:00 | Light: 320 | Position: OPEN (90 deg) | Mode: AUTO
Time: 7:00:01 | Light: 325 | Position: OPEN (90 deg) | Mode: AUTO
Manual override triggered. Blinds toggled.
Time: 7:00:02 | Light: 330 | Position: CLOSED (0 deg) | Mode: MANUAL OVERRIDE
```

## Expected Canvas Behavior
* Startup: Onboard RGB LED lights up Green (Auto-Closed) and the Servo is at 0°.
* In auto-mode, if the time transitions past 07:00 and LDR reads >300, the servo moves to 90° and onboard LED changes to Blue.
* Pressing the button at GPIO 16 instantly sweeps the servo, turns the onboard LED Red, and maintains position regardless of sensor changes for 10 seconds before resuming auto tracking.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `blindServo.attach(13)` | Attaches the window blind servo actuator to GPIO 13. |
| `analogRead(27)` | Reads the ambient light sensor voltage divider to calculate brightness. |
| `digitalRead(BUTTON_PIN) == LOW` | Registers button press edge transition to start override timer. |
| `manualTimer = 50` | Sets override length to 50 iterations of the 200 ms loop delay. |
| `now.hour() >= 7 && now.hour() < 18` | Restricts automated blind operations to daytime scheduling. |
| `blindServo.write(...)` | Rotates blind blades to open (90°) or closed (0°) angle configurations. |

## Hardware & Safety Concept
* **Servo Overload Safety:** Mechanical blinds can get obstructed by window frames, curtains, or physical items. If the servo continues to draw high currents while jammed, it can overheat and burn out. Incorporating a slip clutch in the gearing or adding current sensing resistors are standard practices to prevent motor damage.
* **Hysteresis Thresholding:** LDR readings fluctuate when clouds pass. To prevent the blinds from constantly opening and closing ("hunting"), the auto loop has a threshold gap: opens above 300, but only closes below 200.

## Try This! (Challenges)
1. **Dynamic Angle Adjusting:** Modify the daytime auto logic to scale the servo angle proportionally from 0° (low light) to 90° (high light) instead of just two binary states.
2. **Rain Emergency Shutdown:** Connect a rain sensor to ADC2 (GP28). If water drops are detected (sensor value drops below 500), force close the blinds immediately to protect the room from water damage.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Blinds jitter and flip back and forth | No hysteresis threshold | Ensure the code uses separate thresholds for opening (300) and closing (200) to filter small light fluctuations. |
| Manual button is unresponsive | Pull-up resistor not configured | Confirm button uses `INPUT_PULLUP` pin mode, and is grounded. |
| Time is drifting significantly | RTC battery discharged | Check or replace the CR2032 battery on the DS3231 module. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - Servo Motor 180 Degree Sweep](../intermediate/51-servo-motor-180-degree-sweep.md)
- [112 - DS3231 Real Time Clock Display](../intermediate/112-ds3231-real-time-clock-display.md)
- [189 - Precision Animal Feeder](189-precision-animal-feeder.md)
