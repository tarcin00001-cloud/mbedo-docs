# 189 - Precision Animal Feeder

Build an automated, weight-monitored animal feeder that triggers a feed dispenser (servo motor) at scheduled times from a DS3231 RTC, checks portion size using an HX711 load cell amplifier, and manages the door closure dynamically using a C++ state machine.

## Goal
Learn how to interface an I2C Real-Time Clock (DS3231) and a 24-bit strain gauge amplifier (HX711) simultaneously, coordinate time triggers with weight thresholds, and sweep a servo-driven gate without loops or helper functions.

## What You Will Build
An automatic pet feeding station. The ARIES v3 board tracks the time using a DS3231 RTC. At 08:30:00 (or on manual button press), the servo rotates to 90° to open the dispenser door. The HX711 measures the bowl weight in real-time. Once the weight reaches 50.0 grams, the servo closes the door, returning to 0° to prevent overfeeding.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DS3231 Real Time Clock | `ds3231` | Yes | Yes |
| HX711 Weight Sensor Module | `hx711` | Yes | Yes |
| Load Cell (0-5kg) | `load_cell` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Push Button | `button` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS3231 RTC | VCC | 3V3 | Red | RTC power supply |
| DS3231 RTC | GND | GND | Black | Ground reference |
| DS3231 RTC | SDA | SDA0 (GP17) | Blue | I2C Data line |
| DS3231 RTC | SCL | SCL0 (GP16) | Yellow | I2C Clock line |
| Servo Motor | Signal | GPIO 13 | Yellow | Servo control signal |
| Servo Motor | VCC | 5V | Red | Servo power input |
| Servo Motor | GND | GND | Black | Ground reference |
| HX711 Module | VCC | 5V | Red | Amplifier power input |
| HX711 Module | GND | GND | Black | Ground reference |
| HX711 Module | DOUT | GPIO 14 | Yellow | Weight data output |
| HX711 Module | SCK | GPIO 15 | Green | Weight serial clock |
| Push Button | Pin 1 | GPIO 16 | Orange | Manual override button |
| Push Button | Pin 2 | GND | Black | Ground reference |

> **Wiring tip:** Standard load cells use a wheatstone bridge color convention: Red (E+), Black (E-), Green (A+), White (A-). Double-check your sensor module markings before powering up.

## Code
```cpp
// 189 - Precision Animal Feeder
#include <Wire.h>
#include <RTClib.h>
#include <HX711.h>
#include <Servo.h>

const int SERVO_PIN = 13;
const int HX711_DOUT_PIN = 14;
const int HX711_SCK_PIN = 15;
const int BUTTON_PIN = 16;
const int LED_R_PIN = 23;
const int LED_G_PIN = 24;
const int LED_B_PIN = 25;

Servo gateServo;
HX711 scale;
RTC_DS3231 rtc;

int systemState = 0;          // 0 = IDLE, 1 = DISPENSING, 2 = SUCCESS_COOLDOWN
float targetWeight = 50.0;     // Target food portion in grams
float currentWeight = 0.0;
float calibrationFactor = 420.0;
int lastSecond = -1;
int logTimer = 0;
int feedHour = 8;
int feedMinute = 30;

void setup() {
  Serial.begin(115200);
  Wire.begin();

  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_R_PIN, OUTPUT);
  pinMode(LED_G_PIN, OUTPUT);
  pinMode(LED_B_PIN, OUTPUT);

  // Status LEDs (Active Low on ARIES onboard RGB)
  digitalWrite(LED_R_PIN, HIGH); // OFF
  digitalWrite(LED_G_PIN, LOW);  // ON (Green = IDLE)
  digitalWrite(LED_B_PIN, HIGH); // OFF

  gateServo.attach(SERVO_PIN);
  gateServo.write(0); // Close gate

  scale.begin(HX711_DOUT_PIN, HX711_SCK_PIN);
  scale.set_scale(calibrationFactor);
  scale.tare(); // Zero the scale on boot

  rtc.begin();
  if (rtc.lostPower()) {
    rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
  }

  Serial.println("Precision Animal Feeder Initialized.");
  Serial.println("Scheduled feeding at 08:30:00. Target portion: 50.0g");
}

void loop() {
  DateTime now = rtc.now();

  // Read weight from scale if ready
  if (scale.is_ready()) {
    float rawWeight = scale.get_units(1);
    if (rawWeight < 0.0) {
      rawWeight = 0.0;
    }
    currentWeight = rawWeight;
  }

  // Check scheduled feed time or manual button override (GPIO 16 is Active LOW)
  if (systemState == 0) {
    if ((now.hour() == feedHour && now.minute() == feedMinute && now.second() == 0) || (digitalRead(BUTTON_PIN) == LOW)) {
      systemState = 1; // Start dispensing
      gateServo.write(90); // Open dispenser gate
      digitalWrite(LED_G_PIN, HIGH); // Turn Green LED OFF
      digitalWrite(LED_B_PIN, LOW);  // Turn Blue LED ON (Dispensing status)
      Serial.println("Dispenser gate opened. Dispensing food...");
    }
  }

  // Dispensing logic
  if (systemState == 1) {
    if (currentWeight >= targetWeight) {
      gateServo.write(0); // Close gate
      systemState = 2; // Cooldown state to prevent re-triggering within the same minute
      digitalWrite(LED_B_PIN, HIGH); // Turn Blue LED OFF
      digitalWrite(LED_G_PIN, LOW);  // Turn Green LED ON
      Serial.print("Target reached: ");
      Serial.print(currentWeight, 1);
      Serial.println("g. Dispenser gate closed.");
    }
  }

  // Reset from cooldown state once feed minute passes
  if (systemState == 2) {
    if (now.second() > 5) {
      systemState = 0; // Return to IDLE
    }
  }

  // Periodic serial logging every 1000 ms (5 * 200ms delay)
  logTimer++;
  if (logTimer >= 5) {
    logTimer = 0;
    Serial.print("Time: ");
    Serial.print(now.hour());
    Serial.print(":");
    Serial.print(now.minute());
    Serial.print(":");
    Serial.print(now.second());
    Serial.print(" | Bowl Weight: ");
    Serial.print(currentWeight, 1);
    Serial.print("g / ");
    Serial.print(targetWeight, 1);
    Serial.print("g | State: ");
    if (systemState == 0) Serial.println("IDLE");
    if (systemState == 1) Serial.println("DISPENSING");
    if (systemState == 2) Serial.println("COOLDOWN");
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DS3231 Real Time Clock**, **HX711 Weight Sensor**, **Servo Motor**, and **Push Button** onto the canvas.
2. Wire the RTC: **VCC → 3V3**, **GND → GND**, **SDA → SDA0 (GP17)**, **SCL → SCL0 (GP16)**.
3. Wire the Servo: **Signal → GPIO 13**, **VCC → 5V**, **GND → GND**.
4. Wire the HX711: **VCC → 5V**, **GND → GND**, **DOUT → GPIO 14**, **SCK → GPIO 15**.
5. Wire the Button: **Pin 1 → GPIO 16**, **Pin 2 → GND**.
6. Paste the code into the editor.
7. Select **Interpreted Mode** in the simulation dropdown.
8. Click **Run** to execute the project.
9. Press the Button to manually trigger feeding, or wait for the RTC clock to hit 08:30:00. Adjust the load cell widget's weight slider to 50g and watch the servo close.

## Expected Output
Serial Monitor:
```
Precision Animal Feeder Initialized.
Scheduled feeding at 08:30:00. Target portion: 50.0g
Time: 8:29:58 | Bowl Weight: 0.0g / 50.0g | State: IDLE
Time: 8:29:59 | Bowl Weight: 0.0g / 50.0g | State: IDLE
Dispenser gate opened. Dispensing food...
Time: 8:30:00 | Bowl Weight: 15.3g / 50.0g | State: DISPENSING
Time: 8:30:01 | Bowl Weight: 38.6g / 50.0g | State: DISPENSING
Target reached: 51.2g. Dispenser gate closed.
Time: 8:30:02 | Bowl Weight: 51.2g / 50.0g | State: COOLDOWN
```

## Expected Canvas Behavior
* On startup, the onboard RGB LED glows Green, and the servo is locked at 0°.
* Triggering the feed starts the blue LED and sweeps the servo arm to 90°.
* Adjusting the load cell slider past 50.0g resets the servo to 0° and returns the LED to Green.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `gateServo.attach(13)` | Pin GPIO 13 is registered for PWM servo control signals. |
| `scale.begin(14, 15)` | Sets the digital DOUT and SCK lines on GPIO pins 14 and 15. |
| `scale.tare()` | Resets the baseline load cell weight calculation to zero. |
| `rtc.now()` | Queries the DS3231 RTC for the active timezone time components. |
| `digitalRead(BUTTON_PIN) == LOW` | Checks if the manual override button is pressed. |
| `systemState == 1` | Checks if the system is currently dispensing food. |
| `currentWeight >= targetWeight` | Triggers the safety cutoff once portion threshold is met. |

## Hardware & Safety Concept
* **Load Cell Strain Gauges:** Physical food monitoring prevents food overflow if the feeder chute gets stuck or double-dispenses. Strain gauges alter resistance when the metal bar deforms under food mass, providing feedback through the HX711's 24-bit differential ADC.
* **Servo Stall Mitigation:** Servos draw high stall current if blocked mechanically by large pet kibble. In commercial designs, firmware detects lack of weight changes during dispensing and moves the servo back and forth to clear jams, or cuts power to prevent thermal failure.

## Try This! (Challenges)
1. **Jam Clearance Sequence:** If the scale weight does not increase by at least 2 grams over 3 seconds during dispensing, wiggle the servo between 70° and 110° to clear any kibble blockage.
2. **Double Feeding Protection:** Maintain a cooldown state variable to completely disable manual feeding triggers if the portion target was reached less than 1 hour ago.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not move on trigger | Missing 5V line or incorrect signal pin | Verify the servo signal is on GPIO 13, and powered from 5V, not 3.3V. |
| Weights are wildly inaccurate | Calibration factor incorrect | Calibrate scale using a known physical weight and adjust the `calibrationFactor` parameter. |
| Feeder loops dispensing constantly | Cooldown flag not set | Ensure the `systemState` goes to `2` immediately upon target completion to debounce time checks. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - Servo Motor 180 Degree Sweep](../intermediate/51-servo-motor-180-degree-sweep.md)
- [109 - Analog Scale Reader](../intermediate/109-analog-scale-reader.md)
- [113 - DS3231 RTC Alarm System](../intermediate/113-ds3231-rtc-alarm-system.md)
