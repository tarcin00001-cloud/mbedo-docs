# 82 - ESP32 DS18B20 Temperature Alarm

Monitor temperature with a DS18B20 probe and trigger an emergency audio-visual alert using an LED and a buzzer if the threshold is breached.

## Goal
Learn how to implement high-reliability temperature monitoring loops using digital waterproof sensors for safety cutoff applications.

## What You Will Build
A DS18B20 temperature probe on GPIO 4 monitors a liquid or air environment. If the temperature exceeds a set limit (e.g. 35 °C), an alert LED on GPIO 5 flashes and a buzzer on GPIO 15 sounds a repeating alarm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DS18B20 Temperature Sensor | `ds18b20` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 4.7 kΩ Resistor | `resistor` | No | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS18B20 Sensor | DATA | GPIO4 | Yellow | Temp probe input |
| DS18B20 Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| Pull-up Resistor | VCC to DATA | — | — | Place 4.7k resistor between VCC and DATA |
| LED (Red) | Anode (+) | GPIO5 via 330 Ω | Orange | Alert indicator |
| LED (Red) | Cathode (−) | GND | Black | Ground reference |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Ground reference |

> **Wiring tip:** The DS18B20 DATA pin (GPIO 4) requires a 4.7 kΩ pull-up resistor to 3.3V. Ensure that the active buzzer is connected to GPIO 15 and powered correctly.

## Code
```cpp
// DS18B20 Temperature Alarm
#include <OneWire.h>
#include <DallasTemperature.h>

const int ONEWIRE_BUS = 4;
const int LED_PIN = 5;
const int BUZZER_PIN = 15;

// Safety cutoff temperature limit in Celsius
const float MAX_TEMP_LIMIT = 35.0;

OneWire oneWire(ONEWIRE_BUS);
DallasTemperature sensors(&oneWire);

void setup() {
  Serial.begin(115200);
  
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(LED_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  
  sensors.begin();
  Serial.println("DS18B20 Temperature Alarm Active.");
}

void loop() {
  sensors.requestTemperatures();
  float temp = sensors.getTempCByIndex(0);
  
  if (temp == DEVICE_DISCONNECTED_C) {
    Serial.println("Error: Sensor Disconnected!");
    // Trigger alarm state if sensor is lost during operation (fail-safe)
    digitalWrite(LED_PIN, HIGH);
    digitalWrite(BUZZER_PIN, HIGH);
    delay(500);
    digitalWrite(LED_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);
    delay(500);
    return;
  }
  
  Serial.print("Temp: "); Serial.print(temp, 1); Serial.println(" °C");
  
  // Check threshold
  if (temp > MAX_TEMP_LIMIT) {
    Serial.println("!! OVER TEMPERATURE ALARM !!");
    
    // Fast toggle alarm pattern
    digitalWrite(LED_PIN, HIGH);
    digitalWrite(BUZZER_PIN, HIGH);
    delay(200);
    digitalWrite(LED_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);
    delay(200);
  } else {
    digitalWrite(LED_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);
    delay(1000); // Poll once per second under normal conditions
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DS18B20 Sensor**, **LED**, and **Buzzer** onto the canvas.
2. Wire DATA to **GPIO4**, LED to **GPIO5**, and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Adjust the temperature slider on the DS18B20 above 35 °C to trigger the alarm.

## Expected Output
Serial Monitor:
```
DS18B20 Temperature Alarm Active.
Temp: 24.5 °C
Temp: 32.1 °C
Temp: 36.4 °C
!! OVER TEMPERATURE ALARM !!
```

## Expected Canvas Behavior
* Below 35 °C, the LED and buzzer are OFF.
* Exceeding 35 °C causes the LED to flash and the buzzer to beep rapidly (200ms intervals).
* If the sensor is completely disconnected, the system enters a fail-safe alarm state to warn of connection loss.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `temp == DEVICE_DISCONNECTED_C` | Checks if the sensor connection is lost, triggering a fail-safe alert. |
| `temp > MAX_TEMP_LIMIT` | Evaluates if the current temperature has crossed the 35.0 °C threshold limit. |
| `delay(200)` | Controls the warning beep tempo during the alarm state. |

## Hardware & Safety Concept: Fail-Safe Design
In safety critical designs, systems should always fail into a safe state. If a sensor is disconnected or fails during operation, the control software should detect the anomaly (e.g. checking for Dallas Temperature disconnected values) and immediately trigger an alarm or shut down heating elements, rather than assuming everything is normal.

## Try This! (Challenges)
1. **Under-temperature Freeze Alert**: Add a second blue LED on GPIO 12 that turns on if the temperature drops below 2 °C.
2. **Hysteresis Buffer**: Prevent the alarm from fluttering on/off near the boundary by requiring the temperature to drop below 33 °C before turning the alarm off.
3. **Serial Mute Command**: Read from the Serial Monitor. If the user types "MUTE", silence the buzzer for 10 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm fires continuously even in cold room | Sensor disconnected, reading -127.00 | Check DATA wiring and pull-up resistor; check Serial Monitor for error messages |
| LED flashes but buzzer doesn't make sound | Active/Passive buzzer mix-up | Make sure you are using an active buzzer module |
| Alarm response is slow | Delay in loop is too large | Reduce loop delays or use non-blocking `millis()` timing |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [81 - ESP32 DS18B20 One-Wire Temp Sensor Serial Logs](81-esp32-ds18b20-onewire-temp-sensor-serial-logs.md)
- [83 - ESP32 DS18B20 Multi-Sensor LCD HUD](83-esp32-ds18b20-multi-sensor-lcd-hud.md)
- [72 - ESP32 DHT22 Temperature Alarm](72-esp32-dht22-temperature-alarm.md)
