# 28 - Laser receiver sensor trigger

Detect when a laser security beam is broken using a laser receiver module and sound a buzzer alarm on the VEGA ARIES v3 board.

## Goal
Learn how to interface electro-optical sensors (like laser receivers), understand active-low break-detection logic, and build a light-barrier security alarm.

## What You Will Build
A laser receiver sensor module is aligned with an external laser diode (KY-008 transmitter). The receiver module output is connected to `GPIO 17`. An active buzzer is connected to `GPIO 14`. Under normal conditions, the laser beam strikes the receiver, causing it to output a digital HIGH signal. When an object crosses the path and blocks the beam, the receiver output drops to LOW. The ARIES board detects this transition and sounds the buzzer alarm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Laser Receiver Module | `laser_receiver` (or generic digital input) | Yes | Yes |
| Laser Transmitter Module | `laser_transmitter` (or static laser power) | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Laser Receiver | VCC | 5V (or 3.3V) | Red | Power connection |
| Laser Receiver | GND | GND | Black | Ground connection |
| Laser Receiver | OUT | GPIO 17 | Green | Digital output signal |
| Active Buzzer | Positive (+) | GPIO 14 | Red | Digital control signal |
| Active Buzzer | Negative (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the Laser Receiver VCC and GND pins to the power rails. Wire the receiver's OUT pin to GPIO 17. The Laser Transmitter itself can be powered directly from 5V/GND (through a suitable resistor if required by the model) to emit a constant beam.

## Code
```cpp
// Laser Receiver Sensor Trigger - VEGA ARIES v3
const int LASER_PIN = 17;
const int BUZZER_PIN = 14;

void setup() {
  // Configure laser receiver output pin as digital input
  pinMode(LASER_PIN, INPUT);
  
  // Configure active buzzer pin as output
  pinMode(BUZZER_PIN, OUTPUT);
}

void loop() {
  int laserState = digitalRead(LASER_PIN);
  
  // Laser receiver outputs HIGH when laser beam is detected (unbroken).
  // It outputs LOW when the beam is broken/blocked.
  if (laserState == LOW) {
    digitalWrite(BUZZER_PIN, HIGH); // Sound the alarm buzzer
  } else {
    digitalWrite(BUZZER_PIN, LOW);  // Turn off the buzzer
  }
  
  delay(10); // High polling rate for security break detection
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **Laser Receiver**, a **Laser Transmitter**, and an **Active Buzzer** onto the canvas.
2. Wire the Laser Receiver's **VCC** to **5V**, **GND** to **GND**, and **OUT** pin to **GPIO 17**.
3. Align the Laser Transmitter widget so its beam path targets the Laser Receiver widget. Connect the Transmitter's power pins to **5V** and **GND**.
4. Wire the Active Buzzer's positive terminal (+) to **GPIO 14** and negative terminal (-) to **GND**.
5. Paste the code into the editor.
6. Click **Run**.
7. Click the Laser Transmitter to toggle the laser beam ON. Verify that the buzzer is silent.
8. Block the beam path (e.g. click the obstacle widget or toggle the laser OFF). The buzzer sounds immediately.

## Expected Output
Serial Monitor:
```
System Initialized.
Beam Status: SECURE | Alarm: OFF
Beam Status: BROKEN | Alarm: ON
```

## Expected Canvas Behavior
* The active buzzer sounds and visual audio ripples appear when the laser beam pathway is blocked.
* The buzzer goes silent once the laser beam directly hits the receiver sensor again.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(LASER_PIN, INPUT)` | Configures GPIO 17 as a digital input to read the laser receiver signal. |
| `digitalRead(LASER_PIN)` | Samples the voltage output of the laser receiver. |
| `laserState == LOW` | Checks if the laser beam is broken (output falls to 0V). |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives GPIO 14 HIGH to sound the alarm. |

## Hardware & Safety Concept
* **Laser Receiver Principle**: Non-modulating laser receiver modules use a phototransistor or photoresistor combined with a comparator circuit. When the laser light shines on the sensor, the resistance drops, pulling the output state to HIGH. When the beam is interrupted, the output shifts to LOW.
* **Laser Safety Classifications**: Standard DIY projects use Class 1 or Class 2 lasers (less than 1 mW output power). While they are generally safe, you should never look directly into the laser beam or point it at anyone's eyes. It can cause permanent retinal damage.

## Try This! (Challenges)
1. **Latching Security Alarm**: Modify the code so that once the beam is broken, the alarm remains active (latch ON) until a reset push button on GPIO 16 is pressed.
2. **Beep Rate Warning**: Make the buzzer sound a beep sequence that speeds up when the beam is broken, rather than a solid tone.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer is always ON | Misalignment or Transmitter OFF | Check that the laser emitter is powered and aligned so the red beam directly hits the receiver sensor |
| Buzzer is weak or silent | Voltage too low | Ensure the Active Buzzer is connected to GPIO 14 and that the ARIES board has a stable power supply |
| Constant false triggers | High ambient light | Sunlight or bright room lighting can saturate the receiver. Shield the receiver inside a black tube or straw |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [27 - Vibration sensor alert](27-vibration-sensor-alert.md)
- [29 - Slide switch state reporter](29-slide-switch-state-reporter.md) (Next project)
