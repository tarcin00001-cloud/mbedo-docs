# 72 - ESP32 DHT22 Temperature Alarm

Build a temperature monitor that triggers a flashing LED and a pulsing buzzer alarm when the ambient temperature exceeds a critical threshold.

## Goal
Learn how to apply conditional logic to digital sensor inputs, implementing safety threshold triggers for automated audio-visual alarms.

## What You Will Build
A DHT22 sensor monitors temperature on GPIO 4. If the temperature exceeds a set threshold (e.g. 30 °C), an alert LED on GPIO 5 flashes and an active buzzer on GPIO 15 pulses an emergency warning tone.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Temperature Input |
| DHT22 Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| LED (Red) | Anode (+) | GPIO5 via 330 Ω | Orange | Alert indicator |
| LED (Red) | Cathode (−) | GND | Black | Ground reference |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Ground reference |

> **Wiring tip:** Standard active buzzers generate a sound automatically when driven with a constant logic HIGH. Connect it directly to GPIO 15. The DHT22 requires 3.3V power.

## Code
```cpp
// DHT22 Temperature Alarm
#include <DHT.h>

#define DHTPIN 4
#define DHTTYPE DHT22

const int LED_PIN = 5;
const int BUZZER_PIN = 15;

// Temperature alarm threshold in Celsius
const float TEMP_LIMIT = 30.0; 

DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(115200);
  
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(LED_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  
  dht.begin();
  Serial.println("DHT22 Temperature Alarm ready.");
}

void loop() {
  delay(2000); // 2 second DHT reading interval
  
  float temp = dht.readTemperature();
  
  if (isnan(temp)) {
    Serial.println("Error reading DHT22!");
    return;
  }
  
  Serial.print("Current Temp: "); Serial.print(temp, 1); Serial.println(" °C");
  
  // Check if temperature exceeds limit
  if (temp > TEMP_LIMIT) {
    Serial.println("!! OVER TEMPERATURE WARNING !!");
    
    // Pulsing alarm pattern (runs for about 1 second)
    for (int i = 0; i < 3; i++) {
      digitalWrite(LED_PIN, HIGH);
      digitalWrite(BUZZER_PIN, HIGH);
      delay(150);
      digitalWrite(LED_PIN, LOW);
      digitalWrite(BUZZER_PIN, LOW);
      delay(150);
    }
  } else {
    // Normal operation
    digitalWrite(LED_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, **LED**, and **Buzzer** onto the canvas.
2. Wire the DATA pin of the DHT22 to **GPIO4**, LED to **GPIO5**, and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Slide the temperature on the DHT22 widget above 30 °C to trigger the alarm.

## Expected Output
Serial Monitor:
```
DHT22 Temperature Alarm ready.
Current Temp: 24.5 °C
Current Temp: 28.2 °C
Current Temp: 31.4 °C
!! OVER TEMPERATURE WARNING !!
```

## Expected Canvas Behavior
* While the temperature slider is below 30 °C, the LED and buzzer are off.
* Raising the temperature above 30 °C causes the LED and buzzer to pulse in groups of three beeps.
* The alarm clears once the temperature drops below the threshold.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `temp > TEMP_LIMIT` | Checks if the measured Celsius temperature is higher than the set limit. |
| `for (int i = 0; i < 3; i++)` | Creates a pulsing alert cycle with three flashes and beeps. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives the active buzzer HIGH to make sound. |

## Hardware & Safety Concept: Industrial Thermal Protection Loops
In process automation, temperature alarm systems run as independent control loops. If a system gets too hot, the microchip triggers a safety interlock (e.g. cutting off heaters using a relay, starting exhaust fans, sounding audible warnings). This protects equipment from thermal runaway and prevents fire hazards in battery storage units, engine bays, or server rooms.

## Try This! (Challenges)
1. **Dynamic Hysteresis**: Implement a cooling off boundary (e.g. turn on at 30 °C, but only turn off once it drops below 28 °C) to prevent alarm chatter.
2. **Variable Alarm Frequency**: Increase the alarm beeping speed as the temperature climbs higher above the threshold.
3. **Mute Button**: Integrate a push button on GPIO 12 that silences the buzzer for 60 seconds while keeping the LED flashing.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer clicking but not sounding | Wrong pin assignments or code drives active buzzer with PWM | Ensure digital write is used for active buzzers, not analog/tone |
| Alarm triggers too late | Long delays in the loop | Keep loop delays consistent with sensor read frequencies |
| False alarms triggered at startup | Unstable initial readings | Discard the first reading or wait for the sensor to stabilize in `setup()` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [71 - ESP32 DHT22 Temp & Humidity Serial Logs](71-esp32-dht22-temp-humidity-serial-logs.md)
- [73 - ESP32 DHT22 Temperature LCD HUD](73-esp32-dht22-temperature-lcd-hud.md)
- [116 - ESP32 Multi-Zone Fire System](../expert/116-multi-zone-fire-system.md)
