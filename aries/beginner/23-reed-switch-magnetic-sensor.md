# 23 - Reed switch magnetic sensor

Detect the presence of a magnetic field using a reed switch sensor module and light up an LED indicator on the VEGA ARIES v3 board.

## Goal
Learn how to interface magnetic reed switches, understand proximity sensing applications, and write code to trigger indicators based on magnetic fields.

## What You Will Build
A reed switch is connected to the VEGA ARIES v3 board on pin `GPIO 17` and configured with the internal pull-up resistor. When a magnet is brought close to the switch, the internal magnetic metal contacts close, pulling `GPIO 17` to GND (LOW). An external LED on `GPIO 15` lights up when the magnet is present and turns off when the magnet is removed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Reed Switch Module | `reed_switch` (or generic button switch) | Yes | Yes |
| Magnet | `magnet` (interactive widget) | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Reed Switch | Terminal 1 | GPIO 17 | Green | Digital sense connection |
| Reed Switch | Terminal 2 | GND | Black | Ground connection |
| LED | Anode (+) | GPIO 15 | Red | Digital control signal (through resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** A reed switch behaves like a standard push button switch. By wiring it between GPIO 17 and GND with an internal pull-up enabled, we eliminate the need for external resistors.

## Code
```cpp
// Reed Switch Magnetic Sensor - VEGA ARIES v3
const int REED_PIN = 17;
const int LED_PIN = 15;

void setup() {
  // Configure the reed switch pin as input with internal pull-up
  pinMode(REED_PIN, INPUT_PULLUP);
  
  // Configure the indicator LED pin as output
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int reedState = digitalRead(REED_PIN);
  
  // Under INPUT_PULLUP, LOW means the switch is closed (magnet is close)
  if (reedState == LOW) {
    digitalWrite(LED_PIN, HIGH); // Turn LED ON
  } else {
    digitalWrite(LED_PIN, LOW);  // Turn LED OFF
  }
  
  delay(20); // Small polling interval
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **Reed Switch**, a **Magnet**, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire Terminal 1 of the Reed Switch to **GPIO 17**.
3. Wire Terminal 2 of the Reed Switch to **GND**.
4. Wire the **220 Ω Resistor** between **GPIO 15** and the LED's anode (+). Connect the LED's cathode (-) to **GND**.
5. Paste the code into the editor.
6. Click **Run**.
7. Drag the **Magnet** widget close to the Reed Switch on the canvas and verify that the LED turns ON. Move the magnet away, and the LED turns OFF.

## Expected Output
Serial Monitor:
```
System Initialized.
Magnetic Field: DETECTED (LOW)  | LED: ON
Magnetic Field: UNDETECTED (HIGH)| LED: OFF
```

## Expected Canvas Behavior
* The external LED glows bright red when the magnet widget is dragged close to the reed switch.
* The LED dims when the magnet widget is removed.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(REED_PIN, INPUT_PULLUP)` | Activates internal pull-up on GPIO 17. |
| `digitalRead(REED_PIN)` | Samples the reed switch state. |
| `reedState == LOW` | Condition that checks if the metal reeds have closed due to magnetic attraction. |

## Hardware & Safety Concept
* **How Reed Switches Work**: A reed switch contains two ferromagnetic metal reeds encapsulated in a hermetically sealed glass envelope. In the presence of a magnetic field (from an electromagnet or permanent magnet), the reeds become oppositely magnetized and attract each other, closing the electrical contact. When the magnetic field is removed, the mechanical spring tension of the reeds pulls them apart, opening the circuit.
* **Hermetic Protection**: Because the contacts are sealed inside glass with inert gas, reed switches are protected against corrosion, dust, and humidity, making them excellent for explosive atmospheres or harsh environments (like door and window alarm sensors).

## Try This! (Challenges)
1. **Latching Burglar Alarm**: Modify the code so that once the magnet is removed (e.g. door opened), the LED warning stays ON indefinitely, even if the door is closed again. Add a button on GPIO 16 to reset the alarm state.
2. **Double Beep Alert**: Add an active buzzer on GPIO 14 to emit a rapid double beep only when the magnet is first detected (transitioning from open to closed).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays OFF even when magnet is close | Magnet is too far away or weak | In simulation, drag the magnet widget closer. On hardware, ensure the magnet is within the reed switch's activation range (usually 1–2 cm) |
| LED glows dimly or flickers | Mismatched pins | Make sure the LED resistor is wired to GPIO 15, not GPIO 17 |
| Reed switch stays closed permanently | Magnetized reeds or heavy load | Glass reed switches can weld shut if carrying excessive current. Ensure it only routes signals to high-impedance inputs |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [22 - Limit Switch indicator](22-limit-switch-indicator.md)
- [24 - Button counter (output logs to Serial)](24-button-counter.md) (Next project)
