# 197 - Electric Fence Controller

Build an agricultural electric fence energizer safety node. The system manages pulse relay triggers to energize the perimeter fence, samples an analog current sensor (ACS712) during the pulse window to detect contact events or grounding faults, and sounds a latching alarm if safety violations are detected.

## Goal
Learn how to implement high-speed synchronised measurements (sampling during precise actuator active windows), handle current sensor scaling, build latching fault state machines, and ensure automated safety shutdown sequences without loops or custom helpers.

## What You Will Build
An agricultural smart energizer. The ARIES v3 board controls a high-voltage pulse relay on GPIO 2. The relay is turned ON for 100 ms once every 1.5 seconds. During this 100 ms pulse, the board reads an ACS712 current sensor on ADC0 (GP26). If the current exceeds a threshold (indicating an animal/intruder has touched the fence or a fallen branch has grounded the wire), the system disables further pulses, sounds a buzzer (GPIO 14), and flashes the Warning LED (GPIO 15). The alarm is cleared using a disarm button on GPIO 16.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 5V Relay Module | `relay` | Yes | Yes |
| ACS712 Current Sensor (5A or 20A) | `current_sensor` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |
| Push Button | `button` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| 5V Relay | IN | GPIO 2 | Orange | Pulse relay control |
| 5V Relay | VCC | 5V | Red | Relay coil power |
| 5V Relay | GND | GND | Black | Ground reference |
| ACS712 Sensor | VCC | 5V | Red | Current sensor power |
| ACS712 Sensor | GND | GND | Black | Ground reference |
| ACS712 Sensor | OUT | ADC0 (GP26) | Blue | Analog output voltage |
| Active Buzzer | + | GPIO 14 | Orange | Alarm siren |
| Active Buzzer | - | GND | Black | Ground reference |
| Warning LED | Anode | GPIO 15 | Orange | Fault light |
| Warning LED | Cathode | GND (via 220 Ω) | Black | Current-limiting resistor |
| Push Button | Pin 1 | GPIO 16 | Yellow | Reset disarm button |
| Push Button | Pin 2 | GND | Black | Ground reference |

> **Wiring tip:** The ACS712 sensor outputs a baseline voltage of 2.5V (half of 5V VCC) when zero current is flowing. Current flowing through the terminals shifts this voltage up or down linearly. Ensure all high-voltage sections of the fence circuit are completely isolated from the ARIES board.

## Code
```cpp
// 197 - Electric Fence Controller
#include <Arduino.h>

const int PULSE_RELAY_PIN = 2;
const int CURRENT_SEN_PIN = 26; // ADC0
const int BUZZER_PIN = 14;
const int WARN_LED_PIN = 15;
const int BUTTON_PIN = 16;

// Safety thresholds
const int AMBIENT_ZERO_CURRENT = 512; // 2.5V baseline on 10-bit ADC
const int BREACH_THRESHOLD = 150;     // ADC deviation count indicating contact
const int CYCLE_TICKS = 15;            // 1.5 seconds cycle (15 * 100 ms)

int systemState = 0;   // 0 = Charging, 1 = Pulsing, 2 = Alarm Fault Lockout
int pulseTimer = 0;
int flashTimer = 0;
int maxCurrentDiff = 0;

void setup() {
  Serial.begin(115200);

  pinMode(PULSE_RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(WARN_LED_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  digitalWrite(PULSE_RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(WARN_LED_PIN, LOW);

  Serial.println("Electric Fence Controller Online.");
  Serial.println("Monitoring fence line current pulses...");
}

void loop() {
  // Check disarm button (GPIO 16 is Active LOW)
  if (digitalRead(BUTTON_PIN) == LOW) {
    if (systemState == 2) {
      systemState = 0;
      pulseTimer = 0;
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(WARN_LED_PIN, LOW);
      Serial.println("Fault reset. System re-armed.");
    }
  }

  // State Machine Logic
  if (systemState == 0) {
    // STATE: Charging/Waiting (Relay OFF)
    digitalWrite(PULSE_RELAY_PIN, LOW);
    
    pulseTimer++;
    if (pulseTimer >= CYCLE_TICKS - 1) { // 1.4 seconds elapsed
      systemState = 1;                  // Transition to Pulse state next loop
      maxCurrentDiff = 0;               // Reset current peak reading
    }
  } 
  else if (systemState == 1) {
    // STATE: Pulsing (Relay ON for 100ms)
    digitalWrite(PULSE_RELAY_PIN, HIGH);
    
    // Sample Current Sensor during active pulse window
    int rawVal = analogRead(CURRENT_SEN_PIN);
    int currentDiff = abs(rawVal - AMBIENT_ZERO_CURRENT);
    
    if (currentDiff > maxCurrentDiff) {
      maxCurrentDiff = currentDiff;
    }

    pulseTimer++;
    if (pulseTimer >= CYCLE_TICKS) { // 1.5 seconds reached, pulse completes
      digitalWrite(PULSE_RELAY_PIN, LOW);
      pulseTimer = 0;
      systemState = 0; // Return to Charging

      // Print telemetry log
      Serial.print("Fence Pulse Sent. Peak Deviation: ");
      Serial.print(maxCurrentDiff);
      Serial.println(" ADC steps");

      // Verify safety limits
      if (maxCurrentDiff >= BREACH_THRESHOLD) {
        systemState = 2; // Trip alarm, enter Lockout
        Serial.println("!!! FENCE BREACH DETECTED: UNUSUAL CURRENT DRAW !!!");
        Serial.println("Locking out energizer pulses for safety.");
      }
    }
  } 
  else if (systemState == 2) {
    // STATE: Alarm Fault Lockout
    digitalWrite(PULSE_RELAY_PIN, LOW); // Force relay OFF for safety
    
    // Pulse alarm outputs (100 ms cadence)
    flashTimer++;
    if (flashTimer % 2 == 0) {
      digitalWrite(BUZZER_PIN, HIGH);
      digitalWrite(WARN_LED_PIN, HIGH);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(WARN_LED_PIN, LOW);
    }
  }

  delay(100); // System base clock tick
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Relay**, **ACS712 Current Sensor** (or analog source widget), **Active Buzzer**, **Warning LED**, and **Push Button** onto the canvas.
2. Wire the Relay: **IN → GPIO 2**, **VCC → 5V**, **GND → GND**.
3. Wire the ACS712: **OUT → ADC0 (GP26)**, **VCC → 5V**, **GND → GND**.
4. Wire the Buzzer: **+ → GPIO 14**, **- → GND**; Warning LED: **Anode → GPIO 15**, **Cathode → GND** (via 220 Ω).
5. Wire the Button: **Pin 1 → GPIO 16**, **Pin 2 → GND**.
6. Paste the code into the editor.
7. Select **Interpreted Mode** in the simulation dropdown.
8. Click **Run** to execute.
9. Adjust the current sensor analog source slider. Set the slider near `512` (baseline 2.5V zero current). When the relay widget clicks on, slide the value above `662` or below `362` to simulate contact. Observe the system lockout.

## Expected Output
Serial Monitor:
```
Electric Fence Controller Online.
Monitoring fence line current pulses...
Fence Pulse Sent. Peak Deviation: 5 ADC steps
Fence Pulse Sent. Peak Deviation: 8 ADC steps
Fence Pulse Sent. Peak Deviation: 12 ADC steps
!!! FENCE BREACH DETECTED: UNUSUAL CURRENT DRAW !!!
Locking out energizer pulses for safety.
Fault reset. System re-armed.
```

## Expected Canvas Behavior
* Normal state: The relay widget pulses active green for a split second every 1.5 seconds. Onboard RGB LEDs remain off.
* Breach state: The relay remains permanently inactive. The Warning LED and buzzer flash/beep rapidly in unison.
* Pressing the button on GPIO 16 restores the pulsing cycle and turns off the alarms.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `digitalWrite(PULSE_RELAY_PIN, HIGH)` | Closes the relay contacts to charge the simulated fence capacitors. |
| `analogRead(CURRENT_SEN_PIN)` | Reads current sensor output voltage precisely during the pulse. |
| `abs(rawVal - 512)` | Calculates deviation from the 2.5V zero-current point. |
| `maxCurrentDiff >= BREACH_THRESHOLD` | Evaluates if the current draw is large enough to constitute a breach. |
| `systemState = 2` | Disables future pulses by moving to the lockout state. |
| `digitalWrite(PULSE_RELAY_PIN, LOW)` | Ensures high-voltage elements are isolated when alarms are active. |

## Hardware & Safety Concept
* **Pulse Energizing Safety:** Continuous high-voltage shock can cause heart fibrillation in humans or animals. To be safe and legal, agricultural energizers output extremely short, discrete pulses. A 100 ms pulse followed by a 1.4-second dead time allows anyone touching the wire to safely break contact before the next pulse.
* **Synchronized Measurement:** Readings from ACS712 sensors are only valid when the fence relay is closed and current is flowing. Sampling must be strictly coordinated with the relay active state, ignoring readings during charging states.

## Try This! (Challenges)
1. **Dynamic Voltage Sag Detection:** Read solar battery voltage during pulses. If battery voltage drops by more than 0.5V when the relay pulses, trigger a low-battery service alarm.
2. **Short Circuit vs. Contact Classification:** Classify breaches. A deviation > 300 indicates a dead short circuit (e.g., fence wire cut or touching metal posts), while a deviation between 150 and 300 indicates high-resistance human/animal contact. Output different buzzer cadences for each.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| System flags breaches when fence is clear | Noise or incorrect baseline value | Calibrate `AMBIENT_ZERO_CURRENT` value. Read ADC value when no current is flowing and set it in code. |
| Relay does not cycle | Locked in fault state | Hold GP16 disarm button LOW to clear the alarm state. Check that ACS712 is not outputting noise. |
| High-voltage pulses damage the ARIES board | Ground loops / lack of isolation | Ensure optocoupler isolation is used on the relay module. Connect ARIES ground strictly to sensor grounds, not high-voltage paths. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [53 - DC Motor Start Stop (with Relay)](../intermediate/53-dc-motor-start-stop.md)
- [107 - Fire Warning Station](../intermediate/107-fire-warning-station.md)
- [190 - Laser Security Grid](190-laser-security-grid.md)
