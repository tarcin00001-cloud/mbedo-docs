# 88 - Fire Safety System

Detect fire conditions by simultaneously monitoring temperature (LM35) and smoke/gas (MQ-2), triggering a buzzer alarm and activating a relay to cut power or engage a sprinkler on hazard.

## Goal
Learn how to implement a multi-sensor safety system with dual-channel independent threshold detection and a combined relay-based emergency response.

## What You Will Build
The system continuously reads both the LM35 temperature sensor and the MQ-2 gas sensor.
- If temperature rises above 60 C **or** gas level rises above 600, the system activates:
  - Buzzer alarm (dual-tone siren)
  - Relay closes (simulating a sprinkler or power cutoff circuit)
- When conditions return to normal, the system resets.

**Why A0, A1, D7, and D8?** Pins A0/A1 are analog inputs for LM35 and MQ-2. Pin D7 controls the relay coil. Pin D8 drives the buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| LM35 Temperature Sensor | `lm35_sensor` | Yes | Yes |
| MQ-2 Gas Sensor | `mq2_sensor` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |
| 5V Relay | `relay` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| LM35 Sensor | VCC | 5V | Power supply |
| LM35 Sensor | OUT | A0 | Analog temperature output |
| LM35 Sensor | GND | GND | Ground reference |
| MQ-2 Sensor | VCC | 5V | Power supply |
| MQ-2 Sensor | A0 | A1 | Analog smoke output |
| MQ-2 Sensor | GND | GND | Ground reference |
| Relay | IN | D7 | Control signal |
| Relay | VCC | 5V | Power supply |
| Relay | GND | GND | Ground reference |
| Buzzer | + | D8 | Signal pin |
| Buzzer | - | GND | Ground reference |

## Code
```cpp
const int TEMP_PIN    = A0;
const int GAS_PIN     = A1;
const int RELAY_PIN   = 7;
const int BUZZER_PIN  = 8;

const float TEMP_THRESHOLD = 60.0; // Degrees Celsius
const int   GAS_THRESHOLD  = 600;  // ADC units (0-1023)

bool alarmState = false;

void setup() {
  Serial.begin(9600);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  // Initialize to safe state
  digitalWrite(RELAY_PIN, LOW);
  noTone(BUZZER_PIN);
  
  Serial.println("Fire Safety System Active");
}

void loop() {
  // Read LM35 temperature (10 mV per degree C)
  int rawTemp = analogRead(TEMP_PIN);
  float voltage = (rawTemp / 1023.0) * 5.0;
  float tempC   = voltage * 100.0;
  
  int gasLevel = analogRead(GAS_PIN);
  
  Serial.print("Temp: ");   Serial.print(tempC, 1);  Serial.print(" C | ");
  Serial.print("Gas: ");    Serial.print(gasLevel);
  
  // Check fire conditions (temperature OR smoke threshold exceeded)
  if (tempC > TEMP_THRESHOLD || gasLevel > GAS_THRESHOLD) {
    
    if (!alarmState) {
      alarmState = true;
      Serial.println(" -> FIRE ALARM TRIGGERED");
    }
    
    // Activate relay (sprinkler / power cutoff)
    digitalWrite(RELAY_PIN, HIGH);
    
    // Play alternating siren tones
    tone(BUZZER_PIN, 1200, 300);
    delay(350);
    tone(BUZZER_PIN, 700, 300);
    delay(350);
    
  } else {
    if (alarmState) {
      alarmState = false;
      Serial.println(" -> Conditions Normal. Resetting.");
    } else {
      Serial.println(" -> OK");
    }
    
    // Reset outputs
    digitalWrite(RELAY_PIN, LOW);
    noTone(BUZZER_PIN);
    delay(500);
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **LM35 Sensor**, **MQ-2 Sensor**, **Relay**, and **Buzzer** onto the canvas.
2. Connect LM35: **VCC** to **5V**, **OUT** to **A0**, **GND** to **GND**.
3. Connect MQ-2: **VCC** to **5V**, **A0** to **A1**, **GND** to **GND**.
4. Connect Relay: **IN** to **D7**, **VCC** to **5V**, **GND** to **GND**.
5. Connect Buzzer: **+** to **D8**, **-** to **GND**.
6. Paste the code into the editor.
7. Use the default Arduino interpreted runtime.
8. Click **Run**.
9. Adjust the LM35 temperature slider above 60 C or the MQ-2 slider above 600. Observe the relay clicking and the buzzer siren pattern in the Terminal.

## Expected Output

Terminal (Normal):
```
Fire Safety System Active
Temp: 25.0 C | Gas: 120 -> OK
```

Terminal (Fire Detected):
```
Temp: 72.0 C | Gas: 120 -> FIRE ALARM TRIGGERED
Temp: 72.0 C | Gas: 120 -> FIRE ALARM TRIGGERED
```

## Expected Canvas Behavior

| LM35 Temp | MQ-2 Gas | Relay State | Buzzer Pattern |
| --- | --- | --- | --- |
| < 60 C | < 600 | Open (LOW) | Silent |
| > 60 C | Any | Closed (HIGH) | 1200/700 Hz siren |
| Any | > 600 | Closed (HIGH) | 1200/700 Hz siren |

## Code Walkthrough

| Expression | What It Does |
| --- | --- |
| `voltage * 100.0` | Converts LM35 output voltage to Celsius. The LM35 outputs 10 mV per degree C, so 1V = 100 C. |
| `tempC > TEMP_THRESHOLD \|\| gasLevel > GAS_THRESHOLD` | Logical OR condition: either hazard alone is sufficient to activate the alarm. |
| `alarmState` flag | Prevents the Terminal from printing repeated trigger messages every loop cycle. Only prints state-change events. |

## Hardware & Safety Concept: Fail-Safe and Normally Open vs. Normally Closed
Physical relay modules have two output states:
- **Normally Open (NO)**: The circuit is open (disconnected) at rest and closes only when the relay coil is energized.
- **Normally Closed (NC)**: The circuit is closed (connected) at rest and opens when the relay coil is energized.

For a fire suppression system, you would typically connect the sprinkler solenoid to the **NO** terminal. When fire is detected, the relay closes and triggers the sprinkler.

For a power cutoff (fail-safe), you would connect the load circuit to the **NC** terminal. If the Arduino loses power, the relay opens automatically and cuts the protected circuit — a passive safety default.

## Try This! (Challenges)
1. **3-Zone Alert LED**: Wire three LEDs on D3, D4, D5. Green: everything normal. Yellow: temperature over 45 C (pre-warn). Red: alarm threshold triggered.
2. **Alarm Acknowledge**: Wire a button to D2. After an alarm, pressing the button resets the alarm state and quiets the buzzer for 30 seconds even if readings are still high (simulating human acknowledgment).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temp always reads 0 | LM35 not on A0 | Confirm OUT pin connects to Arduino A0. |
| Relay clicks constantly in normal conditions | Threshold too low | Raise `GAS_THRESHOLD` to 700 or observe raw values in the Terminal and adjust accordingly. |

## Mode Notes
These patterns (dual analog reads, threshold logic, relay control, and buzzer tones) are supported by MbedO interpreted mode.

## Related Projects
- [87 - Air Quality Monitor](87-air-quality-monitor.md)
- [63 - Pressure Alarm](../intermediate/63-pressure-alarm.md)
- [89 - Flood Alert](89-flood-alert.md)
