# 106 - Pico Shake Alarm

Build a latching vibration alarm that sounds a buzzer when a shake is detected and requires a button press to reset.

## Goal
Learn how to capture brief digital vibration sensor pulses, latch alarm states in variables, and process digital reset button events.

## What You Will Build
A security alarm system:
- **Vibration Sensor (GP16)**: Detects physical shocks or vibrations.
- **Active Buzzer (GP14)**: Sounds a continuous alarm if a vibration is detected.
- **Reset Button (GP15)**: Toggles the alarm OFF when pressed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Vibration Sensor Module | `button` | Yes (represented by switch button) | Yes (or SW-420 sensor) |
| Active Buzzer | `buzzer` | Yes | Yes |
| Push Button | `button` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Vibration Sensor | VCC | 3.3V | Power supply |
| Vibration Sensor | OUT | GP16 | Digital shake signal |
| Vibration Sensor | GND | GND | Ground reference |
| Active Buzzer | VCC (+) | GP14 | Sounder control |
| Active Buzzer | GND (-) | GND | Ground return |
| Reset Button | Terminal 1 | GP15 | Reset control pin |
| Reset Button | Terminal 2 | GND | Ground return |

## Code
```cpp
const int VIB_PIN    = 16;
const int RESET_PIN  = 15;
const int BUZZER_PIN = 14;

bool alarmLatched = false;

void setup() {
  pinMode(VIB_PIN, INPUT);
  pinMode(RESET_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(BUZZER_PIN, LOW); // Start silent
}

void loop() {
  int shake = digitalRead(VIB_PIN);
  int reset = digitalRead(RESET_PIN);

  // Active-LOW check: vibration sensors typically pull LOW when contacts slide
  if (shake == LOW) {
    alarmLatched = true; // Lock alarm state
  }

  // Reset button clears the alarm (active-LOW)
  if (reset == LOW) {
    alarmLatched = false;
  }

  // Execute latched alarm state
  if (alarmLatched) {
    // Sound alarm siren
    digitalWrite(BUZZER_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZER_PIN, LOW);
    delay(100);
  } else {
    digitalWrite(BUZZER_PIN, LOW);
  }

  delay(10);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Vibration Sensor** (represented by switch button), **Active Buzzer**, and **Push Button** onto the canvas.
2. Connect Vibration Sensor OUT to **GP16**, Buzzer positive to **GP14**, and Reset Button to **GP15**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Click the vibration switch on canvas to trigger the alarm. Note that it stays active until you click the Reset button.

## Expected Output

Terminal:
```
Simulation active. Latching security alarm online.
```

## Expected Canvas Behavior
* Normal state: Silent.
* Shake Detected (GP16 LOW): Buzzer starts beeping. The beep sequence continues even after the shake stops.
* Reset Pressed (GP15 LOW): Buzzer falls silent immediately.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `alarmLatched = true` | Locks the alarm state in a boolean variable, keeping the buzzer active even after the initial physical vibration has ceased. |

## Hardware & Safety Concept: Alarm Latching Circuits
Vibration sensors (such as the SW-420 spring switch) trigger only for a fraction of a millisecond when bumped. If the code only turned the buzzer ON when the pin was active, the beep would be too short for a user to hear. Latching the alarm state in software ensures the alert remains active until a person manually verifies the site and resets the system.

## Try This! (Challenges)
1. **Visual Alarm LED**: Add a Red LED on GP13 and light it continuously while the alarm is latched.
2. **Double Shake Trigger**: Modify the code so that the alarm only latches if two separate shake events are detected within 2 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers randomly | Sensitivity too high | Adjust the physical potentiometer on the SW-420 sensor module to increase the vibration threshold required to pull OUT LOW. |

## Mode Notes
This basic latch logic project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [05 - Pico Active Buzzer](../../beginner/05-pico-active-buzzer.md)
- [26 - Pico Button Counter](../../beginner/26-pico-button-counter.md)
