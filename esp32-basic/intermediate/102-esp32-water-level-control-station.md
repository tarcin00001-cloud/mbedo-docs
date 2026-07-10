# 102 - ESP32 Water Level Control Station

Build an automated tank pump control station that monitors water level, switching a pump relay to fill the tank when low and triggering a buzzer alert when the tank reaches full capacity.

## Goal
Learn how to implement hysteresis control loops (dual-threshold control) for automatic fluid refills, protecting pump systems from rapid switching.

## What You Will Build
An analog water level sensor is read on GPIO 34. A relay on GPIO 13 controls a water pump, and a buzzer on GPIO 15 acts as a full tank alarm. If the water level drops below the LOW threshold (e.g. < 800), the relay turns ON to start the pump. Once the level reaches the HIGH threshold (e.g. > 2800), the relay shuts OFF, and the buzzer emits a warning chime.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Water Level Sensor Module | `water_sensor` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Water Sensor | SIG (AO) | GPIO34 | Yellow | Fluid level input |
| Water Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| Relay Module | IN (Signal) | GPIO13 | Orange | Pump motor switch |
| Relay Module | VCC / GND | 5V / GND | Red / Black | Power rails |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Warning alarm |
| Active Buzzer | GND (−) | GND | Black | Ground reference |

> **Wiring tip:** The water level sensor outputs a positive analog voltage proportional to the depth of submersion. Connect the SIG pin to GPIO 34. Power the relay and buzzer from the 5V rail.

## Code
```cpp
// Water Level Control Station
const int WATER_PIN = 34;
const int RELAY_PIN = 13;
const int BUZZER_PIN = 15;

// Hysteresis bounds (0 - 4095)
const int LEVEL_LOW = 800;   // Pump starts filling below this level
const int LEVEL_HIGH = 2800; // Pump stops and alarms above this level

bool pumpActive = false;

void setup() {
  Serial.begin(115200);
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("Water Control Station Online.");
}

void loop() {
  int rawLevel = analogRead(WATER_PIN);
  
  Serial.print("Water Level ADC: "); Serial.print(rawLevel);
  Serial.print(" | Pump: "); Serial.println(pumpActive ? "RUNNING" : "STOPPED");
  
  // Hysteresis Loop Logic
  // 1. Tank is Empty -> Start Pump
  if (rawLevel < LEVEL_LOW && !pumpActive) {
    Serial.println(">> TANK LOW: Activating Pump <<");
    digitalWrite(RELAY_PIN, HIGH);
    pumpActive = true;
    
    // Quick double chirp to indicate pump start
    tone(BUZZER_PIN, 1500, 50);
    delay(100);
    tone(BUZZER_PIN, 1500, 50);
  }
  // 2. Tank is Full -> Stop Pump & Alarm
  else if (rawLevel > LEVEL_HIGH && pumpActive) {
    Serial.println(">> TANK FULL: Shutting down Pump <<");
    digitalWrite(RELAY_PIN, LOW);
    pumpActive = false;
    
    // Alert chime (5 long beeps)
    for (int i = 0; i < 5; i++) {
      digitalWrite(BUZZER_PIN, HIGH);
      delay(200);
      digitalWrite(BUZZER_PIN, LOW);
      delay(100);
    }
  }
  
  delay(1000); // Poll once per second
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Water Level Sensor**, **Relay**, and **Buzzer** onto the canvas.
2. Wire Water Sensor SIG to **GPIO34**, Relay to **GPIO13**, and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Adjust the water level slider widget. Move it below the low limit to start the relay, then slide it up past the high limit to shut it off and sound the buzzer.

## Expected Output
Serial Monitor:
```
Water Control Station Online.
Water Level ADC: 420 | Pump: STOPPED
>> TANK LOW: Activating Pump <<
Water Level ADC: 1500 | Pump: RUNNING
Water Level ADC: 2950 | Pump: RUNNING
>> TANK FULL: Shutting down Pump <<
```

## Expected Canvas Behavior
* Starting with a dry sensor slider turns ON the relay (green/active).
* Moving the slider up slowly does not turn off the relay until it crosses the 2800 mark.
* Once the slider crosses 2800, the relay shuts off (grey/inactive) and the buzzer widget flashes.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `rawLevel < LEVEL_LOW && !pumpActive` | Triggers the pump only if it wasn't already running, preventing repeated triggers. |
| `rawLevel > LEVEL_HIGH` | Triggers the high cutoff limit. |
| `tone(BUZZER_PIN, 1500, 50)` | Emits a brief 1500 Hz chime pulse. |

## Hardware & Safety Concept: Hysteresis Control loops (Schmitt Trigger)
If we used a single threshold value (e.g. pump turns on below 1500 and off above 1500), water surface waves or measurement noise would cause the relay to toggle on and off rapidly (chatter). This rapidly destroys relay contacts and burns out pump motors. A **hysteresis loop** solves this by establishing separate activation (800) and deactivation (2800) limits.

## Try This! (Challenges)
1. **Interactive Status LCD**: Add a 16x2 I2C LCD (Project 58) displaying the tank percentage and pump status.
2. **Dry Run Protection**: If the pump is active for more than 10 seconds but the water level does not increase, shut down the pump and flash an error.
3. **Manual Override Jog Button**: Add a push button on GPIO 4 that manually overrides the control loop to run the pump.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay chatters at the threshold | Missing hysteresis logic | Verify separate `LEVEL_LOW` and `LEVEL_HIGH` limits are used |
| Sensor corrodes quickly | Electrolysis | Connect sensor VCC to a GPIO pin and only pull HIGH during reads |
| Pump fails to turn on | Low voltage | Ensure relay power is wired to 5V Vin |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [41 - ESP32 Water Level Analog Level Print](../beginner/41-esp32-water-level-analog-level-print.md)
- [11 - ESP32 Relay Module Control](../beginner/11-esp32-relay-module-control.md)
- [98 - ESP32 Gas Leakage Alarm Node](98-esp32-gas-leakage-alarm-node.md)
