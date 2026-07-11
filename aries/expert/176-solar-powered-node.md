# 176 - Solar Powered Sensor Node

Build a remote environmental monitoring node powered by a solar panel and battery system, featuring voltage divider monitoring, battery level calculation, and adaptive power states.

## Goal
Learn how to read and scale high analog voltages using resistor divider networks on ADC0 and ADC1, monitor ambient light and climate data, and design a dynamic power-management state machine that adjusts sensor polling rates according to battery levels without using loops.

## What You Will Build
A solar-powered telemetry station. The node measures the solar panel output voltage and battery cell voltage. It reads ambient temperature, humidity, and light levels. A state machine evaluates the battery health. If the battery is full, it logs data every 2 seconds. If the battery drops, it enters "Eco Mode" (logging every 10 seconds). If the battery falls below critical levels, it shuts down sensor operations and sounds an alarm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| LDR (Photoresistor) | `ldr` | Yes | Yes |
| Resistor (100 kΩ) | `resistor` | No | Yes (divider) |
| Resistor (33 kΩ) | `resistor` | No | Yes (divider) |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Solar Divider Out | Wiper / Midpoint | ADC0 | Orange | Solar voltage monitoring (GP26) |
| Battery Divider Out| Wiper / Midpoint | ADC1 | Yellow | Battery voltage monitoring (GP27) |
| LDR | OUT (analog) | ADC2 | Blue | Ambient light monitoring (GP28) |
| DHT22 Sensor | DATA | GPIO 12 | Purple | Digital temp/humidity pin |
| DHT22 Sensor | VCC | 3V3 | Red | Power |
| DHT22 Sensor | GND | GND | Black | Ground |
| Active Buzzer | + | GPIO 14 | Gray | Low power alarm |
| Active Buzzer | - | GND | Black | Ground |
| Warning LED | Anode | GPIO 15 | Red | System status indicator |
| Warning LED | Cathode | GND | Black | Ground |

> **Wiring tip:** A resistor voltage divider is critical! Connecting a 12V solar panel directly to ARIES ADC pins will destroy the MCU. A divider with $R_1 = 100\text{ k}\Omega$ and $R_2 = 33\text{ k}\Omega$ scales voltages up to 13.3V down to the safe 3.3V limit of the ARIES analog inputs.

## Code
```cpp
// Solar Powered Sensor Node - VEGA ARIES v3
#include <DHT.h>

const int SOLAR_PIN = ADC0;
const int BATT_PIN = ADC1;
const int LDR_PIN = ADC2;
const int DHT_PIN = 12;
const int BUZZER = 14;
const int LED_PIN = 15;

DHT dht(DHT_PIN, DHT22);

// Calibration Factors
const float SOLAR_SCALE = 4.03; // (100k + 33k) / 33k
const float BATT_SCALE = 2.0;   // (10k + 10k) / 10k (assuming 2:1 divider for 3.7-4.2V LiPo)

unsigned long lastTime = 0;
int sampleInterval = 2000; // 2 seconds default
int powerState = 0; // 0: Normal, 1: Eco, 2: Critical Shutdown

void setup() {
  Serial.begin(115200);
  pinMode(BUZZER, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  dht.begin();
  lastTime = millis();
}

void loop() {
  // Read analog voltages
  int rawSolar = analogRead(SOLAR_PIN);
  int rawBatt = analogRead(BATT_PIN);
  int rawLight = analogRead(LDR_PIN);
  
  // Calculate true voltages
  float solarVolt = ((float)rawSolar * 3.3 / 1023.0) * SOLAR_SCALE;
  float battVolt = ((float)rawBatt * 3.3 / 1023.0) * BATT_SCALE;
  
  // Determine Power State based on Battery Voltage
  if (battVolt >= 3.7) {
    powerState = 0;
    sampleInterval = 2000; // Normal Mode: Poll every 2 seconds
    digitalWrite(LED_PIN, LOW);
  } 
  else if (battVolt < 3.7 && battVolt >= 3.3) {
    powerState = 1;
    sampleInterval = 10000; // Eco Mode: Poll every 10 seconds to save power
    digitalWrite(LED_PIN, HIGH); // Light warning LED continuously
  } 
  else {
    powerState = 2; // Critical Mode: Shutdown telemetry
    sampleInterval = 5000; // Alarm beep frequency
  }
  
  unsigned long currentTime = millis();
  unsigned long elapsed = currentTime - lastTime;
  
  // Trigger sensor readout based on dynamic state interval
  if (elapsed >= sampleInterval) {
    lastTime = currentTime;
    
    if (powerState == 0) {
      // Normal Operation
      float temp = dht.readTemperature();
      float hum = dht.readHumidity();
      
      Serial.print("[NORMAL] Batt: ");
      Serial.print(battVolt);
      Serial.print("V | Solar: ");
      Serial.print(solarVolt);
      Serial.print("V | Light: ");
      Serial.print(rawLight);
      Serial.print(" | Temp: ");
      Serial.print(temp);
      Serial.println("C");
    } 
    else if (powerState == 1) {
      // Eco Mode - telemetry active but reduced rate
      float temp = dht.readTemperature();
      float hum = dht.readHumidity();
      
      Serial.print("[ECO] Batt: ");
      Serial.print(battVolt);
      Serial.print("V | Solar: ");
      Serial.print(solarVolt);
      Serial.print("V | Telemetry rate reduced to 10s.");
      Serial.println();
    } 
    else if (powerState == 2) {
      // Critical Shutdown - Sound alarm, skip DHT22 readings
      Serial.print("[CRITICAL SHUTDOWN] Battery empty: ");
      Serial.print(battVolt);
      Serial.println("V! Disconnecting load.");
      
      // Sound alarm beep
      digitalWrite(BUZZER, HIGH);
      digitalWrite(LED_PIN, HIGH);
      delay(100);
      digitalWrite(BUZZER, LOW);
      digitalWrite(LED_PIN, LOW);
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DHT22**, **LDR**, **Buzzer**, and **LED** components onto the canvas.
2. Feed analog voltage sources into **ADC0** (simulating Solar Panel) and **ADC1** (simulating Battery cell).
3. Paste the code into the editor.
4. Select **Interpreted Mode** and click **Run**.
5. Move the battery simulation slider (ADC1) up and down to observe the console printout and LED/buzzer alarms changing.

## Expected Output
Serial Monitor:
```
[NORMAL] Batt: 4.10V | Solar: 12.14V | Light: 840 | Temp: 28.50C
[NORMAL] Batt: 3.90V | Solar: 5.40V | Light: 250 | Temp: 27.20C
[ECO] Batt: 3.52V | Solar: 0.12V | Light: 10 | Telemetry rate reduced to 10s.
[CRITICAL SHUTDOWN] Battery empty: 3.12V! Disconnecting load.
```

## Expected Canvas Behavior
* With battery voltage slider at maximum (above 3.7V equivalent analog input), the serial console prints normal status messages every 2 seconds.
* Lowering the battery input slider below 3.7V changes the mode to ECO. The warning LED stays ON, and printing slows down to a 10-second frequency.
* Dropping the battery slider below 3.3V causes the buzzer to beep every 5 seconds and prints critical shutdown status.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SOLAR_SCALE` | The resistor ratio scales the ADC reading back to the actual panel terminal voltage. |
| `powerState` | Decides current mode based on battery charge thresholds. |
| `sampleInterval` | Modifies the wait variable dynamically from 2000ms to 10000ms. |
| `elapsed >= sampleInterval` | Executes non-blocking periodic polling using the dynamically updated interval. |
| `powerState == 2` | Disables sensor reads and logs to preserve current during low battery events. |

## Hardware & Safety Concept
* **Reverse Current Protection**: When the sun goes down, the solar panel can draw current from the battery, draining it. Always place a Schottky diode (e.g. 1N5819) between the solar panel and the charging controller to block reverse current.
* **Lithium-Ion Over-Discharge**: Discharging a Lithium-Polymer battery below 3.0V damages the cell permanently. The critical shutdown threshold (3.2V) acts as a software cut-off to prevent battery damage.

## Try This! (Challenges)
1. **Solar Charging Indicator**: Turn on the onboard Green LED (`LED_G` / GP24) only when the solar panel voltage is greater than the battery voltage (charging state).
2. **Dynamic Sleep Simulation**: If the LDR detects total darkness (night) AND solar panel voltage is < 1.0V, increase the polling rate to 30 seconds to conserve battery power.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Voltage readings are completely wrong | Scaling factors match wrong resistor values | Double check your resistor values and adjust BATT_SCALE/SOLAR_SCALE in the code. |
| Node locks up in Critical mode | Integrator/timing errors | Ensure `delay(100)` is only used for the alarm buzzer and doesn't block critical loop logic. |
| Analog reads are noisy | High output impedance of dividers | Add a 0.1uF capacitor in parallel with the lower resistor of the divider to filter noise. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - Automatic Night Light](../intermediate/56-automatic-night-light.md)
- [159 - Automatic Battery Capacity Tester](../advanced/159-automatic-battery-capacity-tester.md)
- [162 - Solar Tracker Efficiency Logger](../advanced/162-solar-tracker-efficiency-logger.md)
