# 56 - Parking Beeper

Build an automotive parking assistant that beeps faster as an obstacle gets closer.

## Goal
Learn how to read distance data and use multi-stage conditional checks to adjust beeping rate intervals dynamically.

## What You Will Build
The Arduino measures distance using the HC-SR04. It adjusts the beeping frequency of the buzzer based on proximity:
- **Safe (> 100 cm)**: Silent.
- **Approaching (50 - 100 cm)**: Slow beeps.
- **Close (15 - 50 cm)**: Fast beeps.
- **Danger (< 15 cm)**: Solid warning tone.

**Why D2, D3, and D8?** Pins D2/D3 manage the distance checks. Pin D8 drives the warning beeper.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-SR04 Ultrasonic | `hc_sr04` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| 100 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 Sensor | VCC | 5V | Power supply (5V) |
| HC-SR04 Sensor | TRIG | D2 | Trigger connection |
| HC-SR04 Sensor | ECHO | D3 | Echo connection |
| HC-SR04 Sensor | GND | GND | Ground reference |
| Buzzer | Pin 1 (Positive) | D8 | Audio control line |
| Buzzer | Pin 2 (Negative) | GND | Ground reference |

## Code
```cpp
const int TRIG_PIN = 2;
const int ECHO_PIN = 3;
const int BUZZ_PIN = 8;

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  Serial.begin(9600);
  Serial.println("Parking Assist Beeper Ready");
}

void loop() {
  // Trigger reading
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH);
  long distance = duration * 0.034 / 2;
  
  Serial.print("Distance: ");
  Serial.print(distance);
  Serial.println(" cm");
  
  // Parking logic based on distance bands
  if (distance > 0 && distance < 15) {
    // 1. Danger Zone: Continuous solid warning tone
    tone(BUZZ_PIN, 1000);
    delay(100);
  } 
  else if (distance >= 15 && distance < 50) {
    // 2. Warning Zone: Fast repeating beep
    tone(BUZZ_PIN, 1000);
    delay(80);
    noTone(BUZZ_PIN);
    delay(80);
  } 
  else if (distance >= 50 && distance < 100) {
    // 3. Approach Zone: Slow repeating beep
    tone(BUZZ_PIN, 1000);
    delay(100);
    noTone(BUZZ_PIN);
    delay(300);
  } 
  else {
    // 4. Safe Zone: Silent
    noTone(BUZZ_PIN);
    delay(200);
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-SR04 Sensor**, and **Buzzer** onto the canvas.
2. Connect HC-SR04 **VCC** to Arduino **5V**, **TRIG** to Arduino **D2**, **ECHO** to Arduino **D3**, and **GND** to Arduino **GND**.
3. Connect Buzzer **pin 1** to Arduino **D8** and **pin 2** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the HC-SR04, adjust the distance slider from 150 cm down to 10 cm, and listen to the beeping speed increase.

## Expected Output

Terminal:
```
Parking Assist Beeper Ready
Distance: 120 cm
Distance: 75 cm
Distance: 32 cm
Distance: 10 cm
...
```

### Expected Canvas Behavior

| Distance Range | Proximity Zone | Buzzer State (D8) | Sound Warning |
| --- | --- | --- | --- |
| > 100 cm | Safe | Silent | None |
| 50 - 100 cm | Approach | Pulsed (100 ms ON, 300 ms OFF) | Slow Beep |
| 15 - 50 cm | Close | Pulsed (80 ms ON, 80 ms OFF) | Fast Beep |
| < 15 cm | Danger | Solid ON (1000 Hz) | Continuous Tone |

The buzzer vibrates and clicks on the canvas faster as the distance slider decreases.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `if (distance > 0 && distance < 15)` | Continuous check for the critical danger zone. |
| `else if (distance >= 15 && distance < 50)` | Medium range check, triggering a fast beep-pause cycle. |
| `noTone(BUZZ_PIN)` | Ensures the buzzer goes silent when no hazard is active or during beep pauses. |

## Hardware & Safety Concept: Sensor Fail-safes
In automotive backup sensors, reliability is paramount. A single bad reading must not cause the vehicle to hit an obstacle. Real-world systems use **rolling averages** (filtering out individual spikes) and check for sensor health. If a sensor gets coated in dirt or ice and returns constant out-of-bounds readings, the system warns the driver that the assist module is temporarily offline.

## Try This! (Challenges)
1. **Pitch Shift**: Modify the code so that the pitch (frequency) of the tone also changes with proximity, becoming higher (e.g. 1500 Hz) in the danger zone.
2. **Visual Warning Meter**: Connect an LED to D13 and make it blink in synchronization with the beeper pulses.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Beeper stays on constantly | Sensor outputting 0 (blind zone or disconnected) | Check that the sensor ECHO pin connects to D3. Verify that the code checks `distance > 0`. |
| Click sound instead of tone | Ground connection missing | Verify Buzzer Pin 2 connects directly to GND. |

## Mode Notes
These patterns (ultrasonic measurements driving conditional beep rate delays) are supported by MbedO interpreted mode.

## Related Projects
- [44 - Obstacle Beeper](44-obstacle-beeper.md)
- [54 - Distance Serial](54-distance-serial.md)
- [55 - Distance Alert LED](55-distance-alert-led.md)
