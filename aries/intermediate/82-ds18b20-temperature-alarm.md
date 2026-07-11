# 82 - DS18B20 Temperature Alarm

Build a temperature alert system using a DS18B20 sensor and an active buzzer that triggers an audible warning when the temperature rises above 30 °C.

## Goal
Learn how to interface the DS18B20 sensor, compare live values against a preset threshold, and switch active buzzed alerts inside a loop-free C++ structure.

## What You Will Build
A DS18B20 temperature sensor is connected to GPIO 12, and an Active Buzzer is connected to GPIO 14 of the ARIES v3 board. If the temperature exceeds 30.0 °C, the buzzer will sound an alarm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DS18B20 Temperature Sensor | `ds18b20` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 4.7 kΩ Resistor (pull-up) | `resistor` | No | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS18B20 Sensor | VCC (Red) | 3V3 | Red | Power |
| DS18B20 Sensor | GND (Black) | GND | Black | Ground reference |
| DS18B20 Sensor | DATA (Yellow) | GPIO 12 | Yellow | OneWire data bus |
| Active Buzzer | Positive (+) | GPIO 14 | Orange | Buzzer control line |
| Active Buzzer | Negative (-) | GND | Black | Ground reference |
| Pull-up Resistor | VCC to DATA | — | — | Place 4.7k resistor between VCC and DATA pins |

> **Wiring tip:** The DS18B20 requires an external 4.7 kΩ pull-up resistor between its DATA pin and VCC (3.3V) for proper OneWire communication. The active buzzer connects directly to GPIO 14.

## Code
```cpp
// DS18B20 Temperature Alarm
#include <OneWire.h>
#include <DallasTemperature.h>

const int ONEWIRE_BUS = 12;
const int BUZZER_PIN = 14;

OneWire oneWire(ONEWIRE_BUS);
DallasTemperature sensors(&oneWire);

void setup() {
  Serial.begin(115200);
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW); // Start with buzzer OFF
  
  sensors.begin();
  Serial.println("DS18B20 Temperature Alarm System Active.");
}

void loop() {
  sensors.requestTemperatures();
  float tempC = sensors.getTempCByIndex(0);
  
  if (tempC == DEVICE_DISCONNECTED_C) {
    Serial.println("Error: Sensor disconnected");
    digitalWrite(BUZZER_PIN, LOW); // Turn off buzzer on error
  } else {
    Serial.print("Temp: ");
    Serial.print(tempC, 1);
    Serial.println(" *C");
    
    // Sound alarm if temperature exceeds 30.0 °C
    if (tempC > 30.0) {
      Serial.println("!! ALARM: Over-temperature Detected !!");
      digitalWrite(BUZZER_PIN, HIGH);
      delay(300);
      digitalWrite(BUZZER_PIN, LOW);
      delay(300);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
      delay(1000); // 1-second delay in normal conditions
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DS18B20 Sensor**, and **Active Buzzer** components onto the canvas.
2. Wire the sensor: **DATA** to **GPIO 12**, **VCC** to **3V3**, and **GND** to **GND**.
3. Wire the buzzer: **Positive (+)** to **GPIO 14** and **Negative (-)** to **GND**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.
7. Slide the temperature slider on the DS18B20 widget above 30 °C and watch the buzzer trigger.

## Expected Output
Serial Monitor:
```
DS18B20 Temperature Alarm System Active.
Temp: 27.5 *C
Temp: 31.4 *C
!! ALARM: Over-temperature Detected !!
```

## Expected Canvas Behavior
* The active buzzer sounds a pulsing alarm when the temperature exceeds 30.0 °C.
* The alarm stops immediately when the temperature slider is pulled below 30.0 °C.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `sensors.begin()` | Initialises the DallasTemperature sensor driver. |
| `sensors.requestTemperatures()` | Commands the DS18B20 to measure temperature. |
| `tempC > 30.0` | Evaluates if the temperature has crossed the alarm threshold. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives GPIO 14 HIGH to sound the buzzer. |

## Hardware & Safety Concept
* **Liquid Temperature Monitoring Safety**: The DS18B20 probe is popular in liquid temperature monitoring. If the liquid gets too hot (for example, in a heating vat or aquarium), an audio alarm notifies operators. The code implements a fail-safe state: if the sensor is disconnected (yielding `-127` °C), the system prints a warning and turns off the buzzer to avoid false alarms, while logging the error.

## Try This! (Challenges)
1. **Freeze & Overheat Alarm**: Modify the code to beep slowly (1 second on/off) if the temperature falls below 5.0 °C, and rapidly if it rises above 30.0 °C.
2. **Warning LED**: Add an LED on GPIO 15 that lights up as a warning at 27.0 °C before the buzzer triggers at 30.0 °C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound | Reversed polarity | Verify the positive buzzer lead is connected to GPIO 14. |
| Temperature is read as -127.00 | Missing pull-up resistor | Connect a 4.7 kΩ pull-up resistor between VCC and DATA. |
| Alarm triggers on startup | Code is reading power-on default | Add a short delay in `setup()` to allow the sensor to stabilize before reading. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [81 - DS18B20 One-Wire Temp Sensor Serial Logs](81-ds18b20-one-wire-temp-serial.md)
- [83 - DS18B20 Multi-Sensor LCD HUD](83-ds18b20-multi-sensor-lcd-hud.md)
- [72 - DHT22 Temperature Alarm](72-dht22-temperature-alarm.md)
