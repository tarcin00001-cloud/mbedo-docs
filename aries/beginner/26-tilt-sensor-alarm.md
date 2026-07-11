# 26 - Tilt sensor alarm

Detect orientation changes using a digital tilt sensor switch and trigger an active buzzer alarm on the VEGA ARIES v3 board.

## Goal
Learn how to interface mechanical tilt sensors (rolling ball switches), configure active-low inputs, and drive an active buzzer as an audible alarm indicator.

## What You Will Build
An SW-520D roll-ball tilt switch is connected to `GPIO 17` and configured with the internal pull-up resistor. An active buzzer is connected to `GPIO 14`. When the sensor is tilted, the internal metallic ball rolls to bridge the contact pins, pulling the input pin to GND (LOW). The VEGA ARIES v3 board detects this tilt and sounds the buzzer alarm. When the sensor is placed upright, the contacts open, and the buzzer stops.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Tilt Switch | `tilt_switch` (or generic button switch) | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Tilt Switch | Terminal 1 | GPIO 17 | Green | Digital sense connection |
| Tilt Switch | Terminal 2 | GND | Black | Ground connection |
| Active Buzzer | Positive (+) | GPIO 14 | Red | Digital control signal |
| Active Buzzer | Negative (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The active buzzer has a polarity (positive longer leg, negative shorter leg). Wire the positive leg to GPIO 14 and the negative leg to GND. Ensure the tilt switch is wired between GPIO 17 and GND.

## Code
```cpp
// Tilt Sensor Alarm - VEGA ARIES v3
const int TILT_PIN = 17;
const int BUZZER_PIN = 14;

void setup() {
  // Configure the tilt sensor pin as input with internal pull-up
  pinMode(TILT_PIN, INPUT_PULLUP);
  
  // Configure the active buzzer pin as output
  pinMode(BUZZER_PIN, OUTPUT);
}

void loop() {
  int tiltState = digitalRead(TILT_PIN);
  
  // Under INPUT_PULLUP, LOW means the contacts are closed (tilted state)
  if (tiltState == LOW) {
    digitalWrite(BUZZER_PIN, HIGH); // Sound the buzzer
  } else {
    digitalWrite(BUZZER_PIN, LOW);  // Turn off the buzzer
  }
  
  delay(50); // Sample rate filtering
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **Tilt Switch** widget, and an **Active Buzzer** onto the canvas.
2. Wire Terminal 1 of the Tilt Switch to **GPIO 17** and Terminal 2 to **GND**.
3. Wire the Active Buzzer's positive terminal (+) to **GPIO 14** and negative terminal (-) to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Click the Tilt Switch widget to tilt/toggle its orientation. The buzzer emits a continuous beep on the canvas. Reset the tilt widget, and the buzzer stops.

## Expected Output
Serial Monitor:
```
System Initialized.
Tilt Sensor: TILTED (LOW)   | Alarm: ON
Tilt Sensor: UPRIGHT (HIGH) | Alarm: OFF
```

## Expected Canvas Behavior
* The active buzzer emits a warning sound and displays visual audio waves on the canvas when the tilt switch is activated (LOW).
* The sound stops when the tilt switch is upright (HIGH).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(TILT_PIN, INPUT_PULLUP)` | Activates internal pull-up on GPIO 17, establishing a default HIGH state. |
| `pinMode(BUZZER_PIN, OUTPUT)` | Defines the buzzer control pin (GPIO 14) as a digital output. |
| `tiltState == LOW` | Checks if the internal metal ball has rolled onto the contacts, pulling the pin to 0V (LOW). |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives GPIO 14 HIGH, sending 3.3V to the active buzzer and sounding the alarm. |

## Hardware & Safety Concept
* **How SW-520D Tilt Switch Works**: An SW-520D is a roll-ball type tilt switch. It consists of a cylinder containing a conductive metal ball and two contact leads at one end. When tilted towards the leads, the ball rolls and creates an electrical bridge, closing the circuit. When tilted away, the ball rolls back, opening the circuit.
* **Active vs Passive Buzzers**: Active buzzers contain an internal oscillator circuit that produces a fixed-frequency sound (typically 2 kHz) as soon as direct current (DC) power is applied. Passive buzzers do not have an internal oscillator and require an alternating AC signal (PWM tone) to produce sound.

## Try This! (Challenges)
1. **Pulsing Alarm**: Modify the code so that when a tilt is detected, the buzzer emits a pulsing beep (200 ms ON, 200 ms OFF) instead of a continuous whine.
2. **Tilt Latch with Reset Button**: Create a latching alarm system. Once tilted, the buzzer must sound continuously until a reset push button on GPIO 16 is pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer makes clicking sound or is silent | Mismatch buzzer type | Ensure you are using an active buzzer widget. Passive buzzers require a PWM signal to sound |
| Buzzer sounds when upright | Inverted mounting | The tilt switch may be mounted upside down. Invert the switch or change the logic to `HIGH` in code |
| Constant whining sound | Ground short | Verify the tilt switch is wired to the correct GPIO 17, and not accidentally bridged to GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [25 - Double click button filter](25-double-click-button-filter.md)
- [27 - Vibration sensor alert](27-vibration-sensor-alert.md) (Next project)
