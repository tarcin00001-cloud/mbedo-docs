# 102 - Pico Water Station

Build an automated water pump controller that refills a container when liquid levels drop.

## Goal
Learn how to use analog level sensors to control high-power pumps via relays and play warning chime alerts.

## What You Will Build
An automatic sump pump controller:
- **Water Level Sensor (GP26)**: Monitors container depth.
- **Relay Module (GP10)**: Activates a 5V pump to refill the tank when the water level drops below the minimum limit.
- **Active Buzzer (GP14)**: Sounds a warning chirp if a critical low level is reached.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Water Level Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Water Sensor | VCC | 3.3V | Power supply |
| Water Sensor | OUT (Analog) | GP26 | Analog depth level |
| Water Sensor | GND | GND | Ground return |
| Relay Module | VCC | 5V | Coil power |
| Relay Module | IN | GP10 | Pump control signal |
| Relay Module | GND | GND | Ground return |
| Active Buzzer | VCC (+) | GP14 | Alarm pin |
| Active Buzzer | GND (-) | GND | Ground return |

## Code
```cpp
const int WATER_PIN  = 26;
const int RELAY_PIN  = 10;
const int BUZZER_PIN = 14;

// Alarm and pump thresholds (raw ADC)
const int DRY_LIMIT  = 800;  // Critical low level
const int FULL_LIMIT = 2800; // Tank full limit

bool pumpActive = false;

void setup() {
  pinMode(WATER_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
}

void loop() {
  int waterLevel = analogRead(WATER_PIN);

  // Dry check: start pump and play alarm
  if (waterLevel < DRY_LIMIT) {
    pumpActive = true;
    digitalWrite(RELAY_PIN, HIGH); // Turn pump ON
    
    // Chirp alert
    digitalWrite(BUZZER_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZER_PIN, LOW);
    delay(100);
  } 
  // Full check: stop pump
  else if (waterLevel > FULL_LIMIT) {
    pumpActive = false;
    digitalWrite(RELAY_PIN, LOW);  // Turn pump OFF
    digitalWrite(BUZZER_PIN, LOW);
  }
  // Intermediate state
  else {
    digitalWrite(BUZZER_PIN, LOW);
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Water Level Sensor** (represented by potentiometer), **Relay Module**, and **Active Buzzer** onto the canvas.
2. Connect Water Sensor to **GP26**, Relay to **GP10**, and Buzzer to **GP14**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the potentiometer to minimum (below 800) to trigger the pump and alarm, then slide it to maximum (above 2800) to stop the pump.

## Expected Output

Terminal:
```
Simulation active. Sump pump automation node online.
```

## Expected Canvas Behavior
| Liquid Level slider | GP10 (Pump Relay) | GP14 (Buzzer) | Pump Status |
| --- | --- | --- | --- |
| Empty (< 800) | HIGH (Closed) | Pulsing | **ON (Refilling + Alarm)** |
| Medium (1500) | HIGH (Closed) | LOW | **ON (Refilling)** |
| Full (> 2800) | LOW (Open) | LOW | OFF (Tank Full) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `waterLevel < DRY_LIMIT` | Detects when the tank has dried out, triggering the fill relay and warning chime. |

## Hardware & Safety Concept: Dry Run Pump Protection
AC water pumps rely on the pumped liquid for lubrication and cooling. Running a pump when the tank is dry (a **dry run**) can overheat the pump coils and damage the seals within minutes. To prevent this, software systems verify that water levels are sufficient before turning the pump ON, and sound alarms if dry conditions are detected.

## Try This! (Challenges)
1. **Indicator Lights**: Wire a green LED on GP15 that turns ON only when the tank is full, and a red LED on GP13 when empty.
2. **Auto Timeout**: Add a safety timer that automatically shuts the pump OFF if it runs continuously for more than 10 seconds, indicating a potential pipe leak.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay switches ON and OFF rapidly | Threshold overlap | Ensure the `FULL_LIMIT` is set significantly higher than the `DRY_LIMIT` to establish a wide operating band. |

## Mode Notes
This multi-device analog control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [11 - Pico Relay Toggle](../../beginner/11-pico-relay-toggle.md)
- [41 - Pico Water Level](../../beginner/41-pico-water-level.md)
