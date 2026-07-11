# 72 - DHT22 Temperature Alarm

Build a temperature alert system that monitors climate data using a DHT22 sensor and triggers an active buzzer warning if the temperature rises above 30 °C.

## Goal
Learn how to check sensor threshold levels inside the main program loop, and activate digital warning peripherals based on sensor state changes.

## What You Will Build
A DHT22 sensor is connected to GPIO 12, and an Active Buzzer is connected to GPIO 14 of the ARIES v3 board. If the detected temperature goes above 30.0 °C, the buzzer turns on to alert the user.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 10 kΩ Resistor (pull-up) | `resistor` | No | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | VCC (Pin 1) | 3V3 | Red | Power input |
| DHT22 Sensor | DATA (Pin 2) | GPIO 12 | Yellow | Bidirectional data line |
| DHT22 Sensor | GND (Pin 4) | GND | Black | Ground reference |
| Active Buzzer | Positive (+) | GPIO 14 | Orange | Buzzer control line |
| Active Buzzer | Negative (-) | GND | Black | Ground reference |
| Pull-up Resistor | VCC to DATA | — | — | Place 10k resistor between VCC and DATA pins |

> **Wiring tip:** The active buzzer connects directly to GPIO 14. If the buzzer makes a faint click or hum without sound, check that the positive (+) pin is connected to GPIO 14 and the negative (-) pin connects to ground.

## Code
```cpp
// DHT22 Temperature Alarm
#include <DHT.h>

#define DHTPIN 12
#define DHTTYPE DHT22

const int BUZZER_PIN = 14;

DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(115200);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW); // Start with buzzer OFF
  
  dht.begin();
  Serial.println("DHT22 Temp Alarm initialized.");
}

void loop() {
  // Query sensor every 2 seconds
  delay(2000);
  
  float temperature = dht.readTemperature(); // Celsius
  
  if (isnan(temperature)) {
    Serial.println("Failed to read from DHT22 sensor!");
    return;
  }
  
  Serial.print("Current Temp: ");
  Serial.print(temperature, 1);
  Serial.println(" *C");
  
  // Trigger active buzzer if temperature is over 30.0 degrees Celsius
  if (temperature > 30.0) {
    Serial.println("!! WARNING: Temperature High !!");
    digitalWrite(BUZZER_PIN, HIGH);
  } else {
    digitalWrite(BUZZER_PIN, LOW);
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DHT22**, and **Active Buzzer** components onto the canvas.
2. Wire the DHT22: **DATA** to **GPIO 12**, **VCC** to **3V3**, and **GND** to **GND**.
3. Wire the Buzzer: **Positive (+)** to **GPIO 14** and **Negative (-)** to **GND**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.
7. Increase the temperature slider past 30 °C on the DHT22 widget and observe the buzzer turning ON.

## Expected Output
Serial Monitor:
```
DHT22 Temp Alarm initialized.
Current Temp: 28.5 *C
Current Temp: 31.2 *C
!! WARNING: Temperature High !!
```

## Expected Canvas Behavior
* The active buzzer sounds and vibrates when the DHT22 temperature slider goes past 30.0 °C.
* Returning the slider below 30.0 °C silences the buzzer immediately on the next loop cycle.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUZZER_PIN, OUTPUT)` | Configures GPIO 14 as a digital output. |
| `dht.readTemperature()` | Reads the temperature value in Celsius. |
| `temperature > 30.0` | Checks if the measured temperature is above the safety threshold limit. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives GPIO 14 HIGH to sound the active buzzer. |

## Hardware & Safety Concept
* **Thermal Alarm Control**: In safety systems, alarm states must latch or refresh continuously to ensure reliability. Thermal monitoring setups typically use a threshold to toggle cooling fans or shut down high-power machinery. In physical designs, adding hysteresis (e.g., turning on at 30 °C and off only when dropping below 29 °C) prevents the buzzer or actuators from rapidly cycling on/off when the temperature fluctuates around the threshold.

## Try This! (Challenges)
1. **Pulse Alarm**: Modify the code so the buzzer beeps repeatedly (100 ms ON, 100 ms OFF) when the alarm is active, instead of staying solid.
2. **Dual-Threshold Indicator**: Add a Warning LED on GPIO 15. Light the LED at 27 °C (Warning level) and turn on the buzzer at 30 °C (Critical level).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound when temp > 30 | Polarity reversed or wrong pin | Verify positive lead connects to GPIO 14 and negative connects to GND. |
| Alarm triggers randomly | Floating threshold | Add a small stabilization average or increase the delay window. |
| Sensor read fails | Wiring error on data | Verify that DHT22 data line is connected to GPIO 12. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [71 - DHT22 Temp & Humidity Serial Logs](71-dht22-temp-humidity-serial.md)
- [73 - DHT22 Temperature LCD HUD](73-dht22-temperature-lcd-hud.md)
- [06 - Active Buzzer Alarm Sequence](../beginner/06-active-buzzer-alarm-sequence.md)
