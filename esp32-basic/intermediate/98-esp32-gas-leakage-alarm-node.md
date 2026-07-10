# 98 - ESP32 Gas Leakage Alarm Node

Build a safety gas detection node that triggers a warning buzzer and activates a relay (to run an exhaust fan or turn off a valve) if gas concentration exceeds safety limits.

## Goal
Learn how to combine analog gas sensors with high-power relay switching and audio alarms to build automated environment safety nodes.

## What You Will Build
An MQ-2 gas sensor analog output is read on GPIO 34. If the gas concentration exceeds a safety threshold (e.g. 1500 raw ADC value), a buzzer on GPIO 15 sounds an alarm, and a relay on GPIO 13 activates (simulating a safety valve shutdown or exhaust fan activation).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MQ-2 Gas Sensor Module | `gas_sensor` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MQ-2 Sensor | AO | GPIO34 | Yellow | Analog concentration |
| MQ-2 Sensor | VCC / GND | 5V / GND | Red / Black | Heater requires 5V |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Ground reference |
| Relay Module | IN (Signal) | GPIO13 | Orange | Exhaust fan control |
| Relay Module | VCC / GND | 5V / GND | Red / Black | Coil power rails |

> **Wiring tip:** The MQ-2 sensor's internal heating element requires **5V** (Vin pin) to function. Connecting VCC to 3.3V will result in flat, zero readings. Make sure to share a common GND between all elements.

## Code
```cpp
// MQ-2 Gas Leakage Alarm Node
const int MQ2_PIN = 34;
const int BUZZER_PIN = 15;
const int RELAY_PIN = 13;

// Safety concentration threshold limit (raw ADC 0-4095)
const int GAS_THRESHOLD = 1500; 

void setup() {
  Serial.begin(115200);
  
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);
  
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(RELAY_PIN, LOW);
  
  Serial.println("Gas Leakage Node Initialising... (30s heater warm-up)");
  delay(5000); // Shortened simulation delay (use 30s in hardware)
  Serial.println("Gas Leakage Node Active.");
}

void loop() {
  int gasVal = analogRead(MQ2_PIN);
  float voltage = gasVal * (3.3f / 4095.0f);
  
  Serial.print("Gas Level ADC: "); Serial.print(gasVal);
  Serial.print(" | Voltage: "); Serial.print(voltage, 2); Serial.println(" V");
  
  // Evaluate gas level
  if (gasVal > GAS_THRESHOLD) {
    Serial.println("!! DANGER: GAS LEAK DETECTED !!");
    
    // Turn on alarm indicators
    digitalWrite(RELAY_PIN, HIGH); // Energize fan/valve relay
    
    // Pulse buzzer alarm
    digitalWrite(BUZZER_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZER_PIN, LOW);
    delay(100);
  } else {
    // Safe environment state
    digitalWrite(RELAY_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);
    delay(1000); // Poll once per second
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MQ-2 Sensor**, **Buzzer**, and **Relay** onto the canvas.
2. Wire MQ-2 AO to **GPIO34**, Buzzer to **GPIO15**, and Relay input to **GPIO13**.
3. Paste the code and click **Run**.
4. Slide the gas concentration value up on the MQ-2 sensor widget. Watch the relay and buzzer activate.

## Expected Output
Serial Monitor:
```
Gas Leakage Node Active.
Gas Level ADC: 450 | Voltage: 0.36 V
Gas Level ADC: 1800 | Voltage: 1.45 V
!! DANGER: GAS LEAK DETECTED !!
```

## Expected Canvas Behavior
* While the gas level slider is below 1500, the relay and buzzer widgets are off.
* Raising the gas slider past 1500 turns ON the relay (green/active) and causes the buzzer to beep rapidly.
* The alarm shuts off when the gas slider is lowered.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `gasVal > GAS_THRESHOLD` | Compares raw concentration inputs against the safety limit. |
| `digitalWrite(RELAY_PIN, HIGH)` | Closes the relay contacts, supplying power to the exhaust fan. |
| `delay(100)` | Sets the beep speed of the alarm outputs. |

## Hardware & Safety Concept: Sensor Warm-up and Cross Sensitivity
MEMS-based gas sensors contain a heating element that must warm up to a stable operating temperature. During the first 1–2 minutes, the readings will spike artificially. Safety systems must ignore these early signals to avoid false alarms.

## Try This! (Challenges)
1. **Air Quality Status LCD**: Integrate an I2C LCD (Project 58) that displays live gas level readings ("Air Quality: OK" / "EVACUATE!").
2. **Latch Alarm mode**: Once triggered, keep the alarm active until a reset button on GPIO 4 is pressed.
3. **Multi-stage warning**: Sound slow beeps at 1000 ADC (Warning), and continuous beeps at 2000 ADC (Emergency).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly at startup | Sensor heater still warming up | Allow the sensor to run for 2-3 minutes to burn off manufacturing residues |
| Relay clicks but buzzer is silent | Pin assignments mismatched | Verify active buzzer is on pin 15 |
| Sensor reads 0 constantly | No 5V power supply | Ensure MQ-2 is wired to the 5V (Vin) pin, not 3.3V |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [43 - ESP32 MQ-2 Gas Sensor Level Print](../beginner/43-esp32-mq2-gas-sensor-level-print.md)
- [11 - ESP32 Relay Module Control](../beginner/11-esp32-relay-module-control.md)
- [92 - ESP32 Bluetooth Relay Switcher](92-esp32-hc05-bluetooth-relay-switcher.md)
