# 104 - ESP32 Night Light Controller

Build an energy-saving automatic night light that activates a lamp relay only when two conditions are met: it is dark (monitored by an LDR) AND human motion is detected (monitored by a PIR).

## Goal
Learn how to implement compound Boolean conditions (AND logic) using multiple input sensors to control a power actuator.

## What You Will Build
An LDR photoresistor is read on GPIO 34, and a PIR motion sensor on GPIO 4. A relay controlling a lamp is connected to GPIO 13. The relay turns ON only when it is dark (LDR < 800) and motion is detected. The light stays on for 10 seconds before checking conditions again.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LDR (Photoresistor) | `ldr` | Yes | Yes |
| HC-SR501 PIR Sensor Module | `pir_sensor` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR | Leg 1 / Leg 2 | 3V3 / GPIO34 | Red / Yellow | Voltage divider (LDR top, 10k bottom) |
| 10 kΩ Resistor | Leg 1 / Leg 2 | GPIO34 / GND | White / Black | Pull-down leg of divider |
| PIR Sensor | OUT | GPIO4 | Yellow | Motion digital input |
| PIR Sensor | VCC / GND | 5V / GND | Red / Black | Power rails |
| Relay Module | IN (Signal) | GPIO13 | Orange | Lamp control |
| Relay Module | VCC / GND | 5V / GND | Red / Black | Coil power rails |

> **Wiring tip:** The LDR and 10 kΩ resistor form a voltage divider to GPIO 34. In bright conditions, the LDR resistance is low and GPIO 34 reads high. In dark conditions, the LDR resistance is high and GPIO 34 reads low.

## Code
```cpp
// Night Light Controller (AND Condition)
const int LDR_PIN = 34;
const int PIR_PIN = 4;
const int RELAY_PIN = 13;

// Light threshold (raw ADC): values below this represent darkness
const int DARK_THRESHOLD = 800; 

void setup() {
  Serial.begin(115200);
  
  pinMode(PIR_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with lamp OFF
  
  Serial.println("Night Light Station ready.");
}

void loop() {
  int ldrVal = analogRead(LDR_PIN);
  bool motion = (digitalRead(PIR_PIN) == HIGH);
  bool isDark = (ldrVal < DARK_THRESHOLD);
  
  Serial.print("LDR: "); Serial.print(ldrVal);
  Serial.print(" | Motion: "); Serial.print(motion ? "YES" : "NO");
  Serial.print(" | Dark: "); Serial.println(isDark ? "YES" : "NO");
  
  // Compound Condition: Dark AND Motion
  if (isDark && motion) {
    Serial.println(">> Condition Met: Turning light ON <<");
    digitalWrite(RELAY_PIN, HIGH); // Turn lamp ON
    
    // Hold the light ON for 10 seconds
    delay(10000); 
    
    Serial.println("Light hold expired. Checking sensor states...");
    digitalWrite(RELAY_PIN, LOW); // Turn OFF to recheck
  } else {
    digitalWrite(RELAY_PIN, LOW);
  }
  
  delay(500); // 2Hz poll rate when inactive
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **LDR**, **PIR Sensor**, and **Relay** onto the canvas.
2. Wire LDR to **GPIO34**, PIR to **GPIO4**, and Relay to **GPIO13**.
3. Paste the code and click **Run**.
4. Slide the LDR widget to dark (low values) AND click the PIR widget to trigger motion. Watch the relay turn ON.
5. If it is bright, triggering motion will do nothing.

## Expected Output
Serial Monitor:
```
Night Light Station ready.
LDR: 3200 | Motion: YES | Dark: NO
LDR: 650 | Motion: NO | Dark: YES
LDR: 580 | Motion: YES | Dark: YES
>> Condition Met: Turning light ON <<
```

## Expected Canvas Behavior
* While LDR slider is in the bright region, the relay widget remains OFF (grey) even if you click the PIR sensor.
* When the LDR slider is dragged to dark AND the PIR sensor is activated, the relay turns ON (green) for 10 seconds.
* The relay turns OFF after 10 seconds unless the conditions are met again.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `ldrVal < DARK_THRESHOLD` | Evaluates if the ambient light drops below the dark threshold. |
| `isDark && motion` | Implements the AND conditional branch. Both variables must be true. |
| `delay(10000)` | Holds the relay ON to give the user time to walk through the room. |

## Hardware & Safety Concept: Energy Efficiency and Sensor Fusion
Smart building controllers use sensor fusion (combining light sensors and motion sensors) to conserve energy. Turning on hallway lights during daytime when people walk past is wasteful; turning on lights at night when nobody is present is also wasteful. Compound conditional control systems ensure energy is spent only when needed.

## Try This! (Challenges)
1. **Interactive Status OLED**: Add an OLED display (Project 60) that shows "ENERGY SAVING ACTIVE" and displays LDR/PIR status.
2. **Dynamic Hold Time**: Connect a potentiometer (GPIO 35) to allow adjusting the light hold time from 5 to 60 seconds.
3. **Double Beep on Trigger**: Add a buzzer on GPIO 15 that plays a warning beep when the light turns on.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Light turns on during daytime | `DARK_THRESHOLD` is set too high | Lower the threshold limit (e.g. 500) so it requires darker rooms to trigger |
| Light never turns on | LDR and resistor swapped | Verify LDR is on the high side (3.3V) and resistor is on the ground side of the divider |
| PIR does not trigger | Cooldown lockout | Wait 3-5 seconds between PIR triggers; ensure the module is fully warmed up |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [34 - ESP32 LDR Automatic Dark Detector](../beginner/34-esp32-ldr-automatic-dark-detector.md)
- [36 - ESP32 PIR Motion Sensor Alert](../beginner/36-esp32-pir-motion-sensor-alert.md)
- [11 - ESP32 Relay Module Control](../beginner/11-esp32-relay-module-control.md)
