# 108 - ESP32 Intrusion Detector Alarm

Build a laser beam-break security barrier that detects intruders, latches a warning relay and buzzer when the beam is broken, and uses a push button to reset.

## Goal
Learn how to construct a laser photoelectric tripwire sensor, detect light level drops (beam break events), and implement latching alarm loops.

## What You Will Build
A laser emitter module (KY-008) is powered by the ESP32. An LDR (photoresistor) acts as the receiver on GPIO 34. A tripwire is created by aligning the laser beam to point directly at the LDR. If an intruder breaks the beam, the LDR value drops. The ESP32 triggers a latching alarm, turning ON a relay on GPIO 13 and a buzzer on GPIO 15. A button on GPIO 4 resets the alarm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| KY-008 Laser Transmitter Module | `laser` | Yes | Yes |
| LDR (Photoresistor) | `ldr` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Laser Module | Pin S (Signal) | GPIO12 | Blue | Turn laser emitter ON/OFF |
| Laser Module | VCC / GND | 5V / GND | Red / Black | Power rails |
| LDR | Leg 1 / Leg 2 | 3V3 / GPIO34 | Red / Yellow | Divider top |
| 10 kΩ Resistor | Leg 1 / Leg 2 | GPIO34 / GND | White / Black | Divider bottom (pull-down) |
| Relay Module | IN (Signal) | GPIO13 | Orange | Warning strobe/siren relay |
| Relay Module | VCC / GND | 5V / GND | Red / Black | Power rails |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Ground reference |
| Push Button | Pin 1 / Pin 2 | 3V3 / GPIO4 | Red / Yellow | Reset switch |
| Resistor (10k) | Leg 1 / Leg 2 | GPIO4 / GND | White / Black | Pull-down for reset button |

> **Wiring tip:** Set up the laser transmitter to point directly at the LDR. In hardware, shield the LDR inside a small dark tube (e.g. a drinking straw) to prevent ambient room light from causing false readings.

## Code
```cpp
// Laser Intrusion Detector Alarm
const int LDR_PIN = 34;
const int LASER_PIN = 12;
const int RELAY_PIN = 13;
const int BUZZER_PIN = 15;
const int RESET_PIN = 4;

// Threshold for laser beam alignment (ADC 0-4095)
// When laser hits LDR directly, raw ADC reading should be > 2500
const int BEAM_THRESHOLD = 1800; // Values below this indicate a beam break

bool alarmActive = false;

void setup() {
  Serial.begin(115200);
  
  pinMode(LASER_PIN, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RESET_PIN, INPUT);
  
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  
  // Turn ON laser emitter
  digitalWrite(LASER_PIN, HIGH);
  
  Serial.println("Security Tripwire System Active.");
}

void loop() {
  int ldrVal = analogRead(LDR_PIN);
  bool resetPressed = (digitalRead(RESET_PIN) == HIGH);
  
  // Tripwire condition: LDR light level drops below threshold
  bool beamBroken = (ldrVal < BEAM_THRESHOLD);
  
  if (beamBroken && !alarmActive) {
    Serial.println("!! ALERT: TRIPWIRE BROKEN !!");
    alarmActive = true;
  }
  
  // Reset logic
  if (resetPressed && alarmActive) {
    Serial.println("Alarm reset by operator.");
    alarmActive = false;
    digitalWrite(RELAY_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);
  }
  
  // Actuate outputs
  if (alarmActive) {
    digitalWrite(RELAY_PIN, HIGH);
    
    // Pulse alarm buzzer
    digitalWrite(BUZZER_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZER_PIN, LOW);
    delay(100);
  } else {
    delay(100); // 10Hz poll rate
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Laser Transmitter**, **LDR**, **Relay**, **Buzzer**, and **Button** onto the canvas.
2. Wire Laser to **GPIO12**, LDR to **GPIO34**, Relay to **GPIO13**, Buzzer to **GPIO15**, and Button to **GPIO4**.
3. Paste the code and click **Run**.
4. Observe the laser widget emitting light towards the LDR widget.
5. Drag the LDR light slider down (simulating beam break). Watch the relay and buzzer latch ON.
6. Click the button widget to reset the alarm.

## Expected Output
Serial Monitor:
```
Security Tripwire System Active.
!! ALERT: TRIPWIRE BROKEN !!
Alarm reset by operator.
```

## Expected Canvas Behavior
* At startup, the laser widget turns ON.
* Sliding the LDR widget light level below 1800 latches the relay ON and pulses the buzzer.
* The alarm stays active even if the LDR is moved back to the bright state.
* Pressing the button widget clears the alarm and turns the relay OFF.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `digitalWrite(LASER_PIN, HIGH)` | Powers the laser transmitter module. |
| `ldrVal < BEAM_THRESHOLD` | Evaluates if the light beam is blocked. |
| `alarmActive = true` | Latches the alarm state in memory. |

## Hardware & Safety Concept: Optical Security Barriers and Anti-Tamper Loops
Laser tripwires provide long-range security detection. By placing mirrors at corner points, a single laser beam can be bounced around an entire perimeter, aligning back to one receiver. If an intruder breaks the beam at any point, the circuit is broken, immediately triggering the alarm.

## Try This! (Challenges)
1. **Intrusion Status OLED Display**: Add an OLED HUD (Project 60) showing "SYSTEM SECURE" or "INTRUSION DETECTED! RESET SYSTEM".
2. **Dynamic Laser Pulse Modulation**: Pulse the laser pin at a specific frequency (e.g. 1 kHz) and verify this frequency on the LDR to prevent intruders from bypassing the sensor with a flashlight.
3. **Tripped Zone indicator**: Use different LDR sensors to log which specific side of the perimeter was breached.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Laser and LDR misaligned | Ensure the laser beam hits the center of the photoresistor |
| Ambient room light prevents alarm trigger | Room is too bright | Shield the LDR inside a small straw or tube to block side light |
| Button doesn't reset alarm | Pin wired to GND without pull-up | Check button pull-down resistor connections on GPIO 4 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [34 - ESP32 LDR Automatic Dark Detector](../beginner/34-esp32-ldr-automatic-dark-detector.md)
- [30 - ESP32 Laser Receiver Signal Detector](../beginner/30-esp32-laser-receiver-signal-detector.md)
- [106 - ESP32 Shake Alarm System](106-esp32-shake-alarm-system.md)
