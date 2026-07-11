# 190 - Laser Security Grid

Secure a room or hallway using a multi-beam laser grid. The system uses three lasers pointing at three Light Dependent Resistors (LDRs) to monitor for intrusion, triggers a continuous latching alarm if any beam is broken, and requires a passcode or physical reset button to disarm.

## Goal
Learn how to monitor multiple analog inputs for threshold events, create a latching alarm state machine, drive optical and audible indicators in phase, and manage system status logs without loops or array structures.

## What You Will Build
A security tripwire matrix. Three laser emitters are wired to GPIO outputs 12, 13, and 10 to keep them energized. Three LDR sensors are placed opposite them, wired to analog pins ADC0 (GP26), ADC1 (GP27), and ADC2 (GP28). If any laser beam is broken, the voltage of the corresponding LDR drops, triggering a latching alarm (buzzer and flashing warning LED) which can only be cleared by pressing the disarm button on GPIO 16.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 3x Laser Diodes (5V/3.3V) | `laser` | Yes | Yes |
| 3x LDR Photoresistors | `ldr` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |
| Push Button | `button` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Laser 1 | VCC | GPIO 12 | Red | Laser power control |
| Laser 1 | GND | GND | Black | Ground reference |
| Laser 2 | VCC | GPIO 13 | Red | Laser power control |
| Laser 2 | GND | GND | Black | Ground reference |
| Laser 3 | VCC | GPIO 10 | Red | Laser power control |
| Laser 3 | GND | GND | Black | Ground reference |
| LDR 1 | Pin 1 | 3V3 | Red | Pull-up source |
| LDR 1 | Pin 2 | ADC0 (GP26) | Blue | Analog output (with 10k resistor to GND) |
| LDR 2 | Pin 1 | 3V3 | Red | Pull-up source |
| LDR 2 | Pin 2 | ADC1 (GP27) | Green | Analog output (with 10k resistor to GND) |
| LDR 3 | Pin 1 | 3V3 | Red | Pull-up source |
| LDR 3 | Pin 2 | ADC2 (GP28) | White | Analog output (with 10k resistor to GND) |
| Active Buzzer | + | GPIO 14 | Orange | Buzzer control |
| Active Buzzer | - | GND | Black | Ground reference |
| Warning LED | Anode | GPIO 15 | Orange | LED control |
| Warning LED | Cathode | GND (via 220 Ω) | Black | Current-limiting resistor |
| Push Button | Pin 1 | GPIO 16 | Blue | Disarm/Reset button |
| Push Button | Pin 2 | GND | Black | Ground reference |

> **Wiring tip:** Standard photoresistors need a voltage divider circuit to read changes. Connect one leg of the LDR to 3.3V, and the other leg to the analog pin (GP26/GP27/GP28). Also connect a 10 kΩ resistor from that same analog pin leg to GND.

## Code
```cpp
// 190 - Laser Security Grid
#include <Arduino.h>

const int LASER1_PIN = 12;
const int LASER2_PIN = 13;
const int LASER3_PIN = 10;

const int LDR1_PIN = 26; // ADC0
const int LDR2_PIN = 27; // ADC1
const int LDR3_PIN = 28; // ADC2

const int BUZZER_PIN = 14;
const int WARN_LED_PIN = 15;
const int BUTTON_PIN = 16;

const int BEAM_THRESHOLD = 400; // Value below this indicates a beam break

int systemArmed = 1;
int alarmActive = 0;
int toggleTick = 0;
int logTimer = 0;

int ldr1Val = 0;
int ldr2Val = 0;
int ldr3Val = 0;

void setup() {
  Serial.begin(115200);

  pinMode(LASER1_PIN, OUTPUT);
  pinMode(LASER2_PIN, OUTPUT);
  pinMode(LASER3_PIN, OUTPUT);

  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(WARN_LED_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  // Turn on lasers
  digitalWrite(LASER1_PIN, HIGH);
  digitalWrite(LASER2_PIN, HIGH);
  digitalWrite(LASER3_PIN, HIGH);

  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(WARN_LED_PIN, LOW);

  Serial.println("Laser Security Grid Online.");
  Serial.println("System Armed. Monitoring beams...");
}

void loop() {
  // Read analog values from each LDR
  ldr1Val = analogRead(LDR1_PIN);
  ldr2Val = analogRead(LDR2_PIN);
  ldr3Val = analogRead(LDR3_PIN);

  // Check reset button (GPIO 16 is Active LOW)
  if (digitalRead(BUTTON_PIN) == LOW) {
    if (alarmActive == 1) {
      alarmActive = 0;
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(WARN_LED_PIN, LOW);
      Serial.println("Alarm disarmed/reset by operator.");
    }
  }

  // Trigger alarm if any beam is broken while armed
  if (systemArmed == 1 && alarmActive == 0) {
    if (ldr1Val < BEAM_THRESHOLD || ldr2Val < BEAM_THRESHOLD || ldr3Val < BEAM_THRESHOLD) {
      alarmActive = 1;
      Serial.println("!!! SECURITY BREACH DETECTED !!!");
      if (ldr1Val < BEAM_THRESHOLD) Serial.println("Breach Source: Beam 1 (GP26)");
      if (ldr2Val < BEAM_THRESHOLD) Serial.println("Breach Source: Beam 2 (GP27)");
      if (ldr3Val < BEAM_THRESHOLD) Serial.println("Breach Source: Beam 3 (GP28)");
    }
  }

  // Alarm flashing logic
  if (alarmActive == 1) {
    toggleTick++;
    if (toggleTick % 2 == 0) {
      digitalWrite(BUZZER_PIN, HIGH);
      digitalWrite(WARN_LED_PIN, HIGH);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(WARN_LED_PIN, LOW);
    }
  }

  // Periodic Logging (every ~1000 ms based on 200 ms loop delay)
  logTimer++;
  if (logTimer >= 5) {
    logTimer = 0;
    Serial.print("Beam 1: ");
    Serial.print(ldr1Val);
    Serial.print(" | Beam 2: ");
    Serial.print(ldr2Val);
    Serial.print(" | Beam 3: ");
    Serial.print(ldr3Val);
    Serial.print(" | Status: ");
    if (alarmActive == 1) {
      Serial.println("ALARM ACTIVE");
    } else {
      Serial.println("ARMED & SECURE");
    }
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **3x Lasers**, **3x LDRs**, **Active Buzzer**, **Warning LED**, and **Push Button** onto the canvas.
2. Wire the Lasers: VCC to **GPIO 12**, **GPIO 13**, **GPIO 10** respectively; GND to **GND**.
3. Wire the LDRs: VCC to **3V3**; outputs to **GP26**, **GP27**, **GP28** respectively (with 10k pull-down resistors to GND).
4. Wire the Buzzer: **+ → GPIO 14**, **- → GND**.
5. Wire the Warning LED: **Anode → GPIO 15**, **Cathode → GND** (via 220 Ω).
6. Wire the Button: **Pin 1 → GPIO 16**, **Pin 2 → GND**.
7. Paste the code into the editor.
8. Select **Interpreted Mode** in the simulation dropdown.
9. Click **Run** to execute.
10. Trigger a breach by moving the slider of any LDR widget below 400 (simulating beam obstruction). Press the disarm button to clear the alert.

## Expected Output
Serial Monitor:
```
Laser Security Grid Online.
System Armed. Monitoring beams...
Beam 1: 850 | Beam 2: 890 | Beam 3: 840 | Status: ARMED & SECURE
Beam 1: 848 | Beam 2: 892 | Beam 3: 842 | Status: ARMED & SECURE
!!! SECURITY BREACH DETECTED !!!
Breach Source: Beam 2 (GP27)
Beam 1: 850 | Beam 2: 120 | Beam 3: 840 | Status: ALARM ACTIVE
```

## Expected Canvas Behavior
* At runtime, the three lasers light up. The buzzer remains quiet and the Warning LED is OFF.
* Decreasing any LDR reading below 400 immediately initiates a rapid alternating flash on the Warning LED and beeping from the buzzer.
* The alarm latches (remains active) even if the LDR level returns to normal, until the disarm button on GPIO 16 is held down.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `digitalWrite(LASER1_PIN, HIGH)` | Energizes the laser emitter to project a light beam onto the LDR. |
| `ldr1Val = analogRead(26)` | Reads raw ADC counts (0–1023) corresponding to Beam 1 LDR voltage. |
| `ldr1Val < BEAM_THRESHOLD` | Detects a drop in received light, signaling a physical beam disruption. |
| `alarmActive = 1` | Latches the alarm status block, ignoring later LDR increases. |
| `digitalRead(BUTTON_PIN) == LOW` | Verifies disarm button is pressed, resetting the state to normal. |
| `toggleTick % 2 == 0` | Standard modulo condition creating a pulsing alert cadence. |

## Hardware & Safety Concept
* **Photoresistor Voltage Divider:** LDRs act as variable resistors whose resistance decreases as light level increases. By placing them in series with a 10 kΩ resistor, the voltage at the ADC pin varies from ~0V (complete darkness) to ~3V3 (direct laser exposure).
* **Latching Alarm Concept:** A basic security system must always use *latching* alert logic. If an intruder quickly walks through a beam, the sensor will only read dark for a split second. A latching state machine records the breach in memory, preventing the alarm from turning off when the intruder moves past the beam.

## Try This! (Challenges)
1. **Calibration Mode:** Write logic to measure and print ambient light levels during the first 3 seconds of boot, dynamically setting the threshold to 70% of the initial ambient reading.
2. **Beep-Out Armed Indicator:** Chirp the buzzer briefly twice every 10 seconds to indicate that the security system is armed and active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers immediately on boot | Lasers misaligned or threshold too high | Check that lasers are active. Verify alignment with LDRs. Reduce `BEAM_THRESHOLD` to 300. |
| Button does not disarm the alarm | Internal pull-up not active | Ensure `INPUT_PULLUP` is set in setup, and button is wired to GND. |
| Buzzer sounds very faint | GPIO overloading | Verify you are using an Active Buzzer (contains its own oscillator) and not a passive speaker. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [108 - Intrusion Detector Alarm](../intermediate/108-intrusion-detector-alarm.md)
- [188 - Earthquake Early Warning](188-earthquake-early-warning.md)
