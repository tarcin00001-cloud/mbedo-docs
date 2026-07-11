# 22 - Limit Switch indicator

Interface a mechanical limit switch (microswitch) to trigger a warning LED indicator on the VEGA ARIES v3 board.

## Goal
Learn how to interface mechanical limit switches, understand active-low input wiring using internal pull-up resistors, and control an output device to signal physical bounds.

## What You Will Build
A limit switch is wired between `GPIO 17` and `GND`. When the switch's metal lever is pressed (actuated), the internal contacts close, pulling `GPIO 17` to GND (LOW). An external LED on `GPIO 15` lights up to signal that a physical limit has been reached, acting as a warning indicator.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Limit Switch | `limit_switch` (or generic button switch) | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Limit Switch | COM (Common) | GND | Black | Ground connection |
| Limit Switch | NO (Normally Open) | GPIO 17 | Green | Digital sense connection |
| LED | Anode (+) | GPIO 15 | Red | Digital control signal (through resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the COM terminal of the limit switch to ARIES GND, and the NO (Normally Open) terminal to GPIO 17. The NC (Normally Closed) terminal is left disconnected. The internal pull-up on GPIO 17 keeps the line HIGH until contact is closed.

## Code
```cpp
// Limit Switch Indicator - VEGA ARIES v3
const int SWITCH_PIN = 17;
const int LED_PIN = 15;

void setup() {
  // Configure the limit switch pin as input with internal pull-up
  pinMode(SWITCH_PIN, INPUT_PULLUP);
  
  // Configure the warning LED pin as output
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int switchState = digitalRead(SWITCH_PIN);
  
  // Under INPUT_PULLUP, LOW means the switch is closed (lever pressed)
  if (switchState == LOW) {
    digitalWrite(LED_PIN, HIGH); // Warning LED ON
  } else {
    digitalWrite(LED_PIN, LOW);  // Warning LED OFF
  }
  
  delay(20); // Basic polling delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **Limit Switch** widget, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire the limit switch's **COM** terminal to **GND** and **NO** terminal to **GPIO 17**.
3. Wire the **220 Ω Resistor** between **GPIO 15** and the LED's anode (+). Connect the LED's cathode (-) to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Click the limit switch lever widget on the canvas. The warning LED lights up immediately. Release the lever, and the LED turns OFF.

## Expected Output
Serial Monitor:
```
System Initialized.
Limit Switch: ACTIVE (LOW)  | Warning LED: ON
Limit Switch: INACTIVE (HIGH) | Warning LED: OFF
```

## Expected Canvas Behavior
* The warning LED lights up when the limit switch lever is depressed and turns OFF when released.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(SWITCH_PIN, INPUT_PULLUP)` | Activates the internal pull-up resistor on GPIO 17, establishing a HIGH logic level while contacts are open. |
| `digitalRead(SWITCH_PIN)` | Samples the limit switch contact state. |
| `switchState == LOW` | Checks if the COM and NO contacts have closed, routing the GPIO pin directly to GND (LOW). |

## Hardware & Safety Concept
* **Limit Switch Wiring Modes**: Mechanical microswitches have three terminals: COM, NO (Normally Open), and NC (Normally Closed). 
  * **Normally Open (NO)**: The circuit is open (disconnected) until the lever is pressed. We use this configuration to detect collisions or boundaries.
  * **Normally Closed (NC)**: The circuit is closed (connected) by default, and opens when the lever is pressed. NC is often preferred in critical safety systems (like industrial E-stops) because if a wire breaks, the system immediately registers an open-circuit fault condition.
* **Mechanical Bounce**: Metal levers bounce mechanically when closing. While our 20 ms delay handles simple bounce, industrial systems combine hardware filters (RC networks) with code logic to guarantee faultless operation.

## Try This! (Challenges)
1. **Change to NC Logic**: Move the wire from NO to the NC terminal of the limit switch and rewrite the code logic so that the LED functions identically (lights up when pressed).
2. **Latched Crash Alarm**: Modify the code so that once the limit switch is hit, the LED stays ON permanently until the system is reset (simulating a crash state that requires manual clearing).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays ON when switch is not pressed | Wired to NC instead of NO | Verify that the jumper wire is connected to the Normally Open (NO) terminal of the switch |
| LED never turns ON | Incorrect input mode | Ensure `INPUT_PULLUP` is used, otherwise the pin floats and may not register a LOW state |
| Flickering status | Bad ground connection | Make sure the COM terminal is securely tied to ARIES GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [21 - Touch Sensor Toggle switch](21-touch-sensor-toggle-switch.md)
- [23 - Reed switch magnetic sensor](23-reed-switch-magnetic-sensor.md) (Next project)
