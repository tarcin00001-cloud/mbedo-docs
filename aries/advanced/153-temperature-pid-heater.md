# 153 - Temperature PID Heater Control

Maintain a precise target temperature using PID feedback from a DHT22 climate sensor to control a relay-based heating element via software PWM.

## Goal
Learn how to implement a time-proportioned software Pulse Width Modulation (slow-PWM) PID loop in C++ to regulate thermal processes without mechanical relay fatigue or code loops.

## What You Will Build
A closed-loop temperature control station. A DHT22 sensor reads the ambient temperature. The PID equation calculates the discrepancy from the setpoint temperature (e.g., 37.0°C). It translates the output into a time-proportioned duty cycle over a 2-second window. The relay on GPIO 15 turns ON and OFF for fractions of this window to power the heating element and maintain temperature stability.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Climate Sensor | `dht22` | Yes | Yes |
| 5V Relay Module | `relay` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | VCC | 3V3 | Red | Power (3.3V compatible) |
| DHT22 Sensor | GND | GND | Black | Ground reference |
| DHT22 Sensor | SDA (Data) | GPIO 12 | Green | Single-wire data line |
| Relay Module | VCC | 5V | Red | Power supply (5V) |
| Relay Module | GND | GND | Black | Ground reference |
| Relay Module | IN (Signal) | GPIO 15 | Blue | Control signal from ARIES |

> **Wiring tip:** GPIO 15 on the VEGA ARIES v3 board is also mapped to the Warning LED. Ensure no hardware conflicts arise if an external relay module is connected to this pin.

## Code
```cpp
// Temperature PID Heater Control - VEGA ARIES v3
#include <DHT.h>

const int DHT_PIN = 12;
#define DHTTYPE DHT22

const int RELAY_PIN = 15;

DHT dht(DHT_PIN, DHTTYPE);

// Target Temperature in Celsius
float targetTemp = 37.0;

// PID Tuning Constants
float Kp = 50.0;  // Proportional gain
float Ki = 2.0;   // Integral gain
float Kd = 10.0;  // Derivative gain

float lastError = 0.0;
float integral = 0.0;

unsigned long lastPIDTime = 0;
unsigned long windowStartTime = 0;
const unsigned long WINDOW_SIZE = 2000; // 2-second software PWM window
unsigned long dutyCycleMs = 0;          // Time relay remains active (0 to WINDOW_SIZE)

void setup() {
  Serial.begin(115200);
  dht.begin();
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);

  lastPIDTime = millis();
  windowStartTime = millis();
}

void loop() {
  float currentTemp = dht.readTemperature();

  // Only run PID calculations if the sensor read succeeds
  if (!isnan(currentTemp)) {
    unsigned long currentTime = millis();
    unsigned long elapsed = currentTime - lastPIDTime;

    // Recalculate PID every 1000ms
    if (elapsed >= 1000) {
      float elapsedSec = (float)elapsed / 1000.0;
      lastPIDTime = currentTime;

      float error = targetTemp - currentTemp;
      
      // Prevent integral windup: only accumulate if close to target
      if (error > -5.0 && error < 5.0) {
        integral = integral + (error * elapsedSec);
      } else {
        integral = 0.0;
      }

      float derivative = (error - lastError) / elapsedSec;
      lastError = error;

      float pidOutput = (Kp * error) + (Ki * integral) + (Kd * derivative);

      // Translate PID output to millisecond active duty time
      if (pidOutput <= 0.0) {
        dutyCycleMs = 0;
      } else if (pidOutput >= 100.0) {
        dutyCycleMs = WINDOW_SIZE; // 100% duty cycle
      } else {
        dutyCycleMs = (unsigned long)((pidOutput / 100.0) * WINDOW_SIZE);
      }

      Serial.print("Temp: ");
      Serial.print(currentTemp, 1);
      Serial.print(" C | Error: ");
      Serial.print(error, 1);
      Serial.print(" | Duty Cycle: ");
      Serial.print((float)dutyCycleMs / WINDOW_SIZE * 100.0, 0);
      Serial.println("%");
    }
  }

  // Manage software PWM duty window
  unsigned long now = millis();
  if (now - windowStartTime >= WINDOW_SIZE) {
    windowStartTime = now; // Shift window start time
  }

  if (now - windowStartTime < dutyCycleMs && dutyCycleMs > 0) {
    digitalWrite(RELAY_PIN, HIGH); // Turn heater ON
  } else {
    digitalWrite(RELAY_PIN, LOW);  // Turn heater OFF
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DHT22 Climate Sensor**, and **5V Relay Module** onto the canvas.
2. Connect the DHT22: **VCC** to **3V3**, **GND** to **GND**, and **SDA** to **GPIO 12**.
3. Connect the Relay: **VCC** to **5V**, **GND** to **GND**, and **IN** to **GPIO 15**.
4. Copy the project code and paste it into the editor.
5. Click **Run**.
6. Use the DHT22 widget's temperature slider to vary the current reading above and below 37.0°C. Observe the relay clicks and LED status indicator changing.

## Expected Output
Serial Monitor:
```
Temp: 32.5 C | Error: 4.5 | Duty Cycle: 100%
Temp: 35.8 C | Error: 1.2 | Duty Cycle: 72%
Temp: 36.9 C | Error: 0.1 | Duty Cycle: 15%
Temp: 37.2 C | Error: -0.2 | Duty Cycle: 0%
```

## Expected Canvas Behavior
* If the DHT22 temperature is below 32°C, the relay switches permanently ON (100% duty).
* As temperature approaches the 37°C setpoint, the relay starts clicking ON and OFF periodically.
* When temperature goes above 37°C, the relay stays OFF.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `dht.readTemperature()` | Captures current thermal readings from the DHT22. |
| `elapsed >= 1000` | Updates PID calculations once per second to prevent overshooting. |
| `error = targetTemp - currentTemp` | Computes heating deficit. |
| `integral = integral + ...` | Accumulates long-term offset errors. |
| `dutyCycleMs = ...` | Scales the 0-100 PID percentage to milliseconds out of `WINDOW_SIZE`. |
| `now - windowStartTime < dutyCycleMs` | High state timer logic for the slow software PWM control. |

## Hardware & Safety Concept
* **Relay Lifecycle Considerations**: Mechanical relays have a finite lifespan (typically 100,000 operations). Switching them at standard PWM frequencies (hundreds of Hz) will destroy them within hours. Software PWM with a long window (2 seconds) protects the contacts.
* **Thermal Runaway Protection**: If the DHT22 fails, the code reads `NaN`. The logic checks `!isnan()` before applying the PID output. In real applications, add a hardware thermal fuse to prevent runaway heating if software hangs.

## Try This! (Challenges)
1. **Dynamic Setpoint**: Wire a potentiometer to `ADC0` and map the target temperature setpoint between 20°C and 50°C.
2. **Cooling Mode**: Modify the software logic to implement a cooling PID loop where the relay controls a cooling fan when temperature exceeds a target threshold.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay does not switch ON when temperature is low | PID limits saturated or faulty wiring | Verify that the Relay IN pin is wired to ARIES GPIO 15 and that the DHT22 slider is set below 37°C. |
| Relay switches continuously and rapidly | `WINDOW_SIZE` value too small | Increase `WINDOW_SIZE` to 5000 (5 seconds) to slow down the physical click frequency. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [10 - Relay Module Switch](../beginner/10-relay-module-switch.md)
- [72 - DHT22 Temperature Alarm](../intermediate/72-dht22-temperature-alarm.md)
