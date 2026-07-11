# 10 - Relay Module Switch

Control an external electromagnetic relay module using a digital output pin on the VEGA ARIES v3 board.

## Goal
Learn how to control high-power external loads using digital pins, understand the electrical separation of low-voltage logic from high-voltage load lines, and configure digital output registers.

## What You Will Build
An electromagnetic relay module connected to pin `GPIO 15` is toggled ON and OFF every 2000 milliseconds. When the relay is energized, its mechanical switch clicks closed, connecting a secondary high-power circuit (simulated by a light bulb or load).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 5V Relay Module | `relay` | Yes | Yes |
| External load (e.g. Lamp) | `lamp` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Relay Module | VCC (+) | 5V | Red | Power supply for relay coil (5V) |
| Relay Module | GND (-) | GND | Black | Ground connection |
| Relay Module | IN (Signal) | GPIO 15 | Orange | Digital control signal |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the relay module's VCC pin to the VEGA ARIES 5V power pin rather than 3V3, as most relay coils require 5V to switch mechanically.

## Code
```cpp
// Relay Module Switch - VEGA ARIES v3
const int RELAY_PIN = 15;

void setup() {
  // Configure the relay control pin as a digital output
  pinMode(RELAY_PIN, OUTPUT);
}

void loop() {
  // Energize the relay coil (turns switch ON)
  digitalWrite(RELAY_PIN, HIGH);
  delay(2000);
  
  // De-energize the relay coil (turns switch OFF)
  digitalWrite(RELAY_PIN, LOW);
  delay(2000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **Relay Module**, and a **Lamp** onto the canvas.
2. Wire the Relay's IN pin to **GPIO 15**, VCC to **5V**, and GND to **GND**.
3. Wire the Lamp through the Relay's Normally Open (NO) and Common (COM) contacts to an external power supply.
4. Paste the code into the editor.
5. Click **Run**.
6. Observe the Relay widget clicking and the Lamp turning ON and OFF every 2 seconds.

## Expected Output
Serial Monitor:
```
System Initialized.
Relay Coil: ENERGIZED (Normally Open contacts closed)
Relay Coil: DE-ENERGIZED (Normally Open contacts open)
```

## Expected Canvas Behavior
* The Relay switch flips position on the canvas, and the connected Lamp lights up for 2 seconds, goes dark for 2 seconds, and repeats.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(RELAY_PIN, OUTPUT)` | Configures pin 15 as a digital output to control the relay module's input transistor. |
| `digitalWrite(RELAY_PIN, HIGH)` | Drives the signal HIGH, energizing the relay coil. |
| `delay(2000)` | Keeps the relay in its active state for 2 seconds. |

## Hardware & Safety Concept: Electromagnetic Isolation and Flyback Diodes
* **Electromagnetic Isolation**: Relays provide physical isolation (air gap) between low-voltage control logic (3.3V/5V) and high-voltage AC/DC power loads. This prevents high voltage from reaching the microcontroller.
* **Flyback Diodes**: A relay coil is an inductor. When the control transistor turns OFF, the collapsing magnetic field creates a high-voltage spike (flyback voltage). Ensure your relay module includes an onboard flyback diode to clamp this spike and protect the microcontroller.

## Try This! (Challenges)
1. **Pulse switching**: Shorten the ON time to 500 milliseconds and the OFF time to 5 seconds to simulate a pulsing solenoid control cycle.
2. **Indicator Sync**: Wire an external LED to GPIO 13 and modify the code so the LED turns ON when the relay is active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay does not click | Insufficient coil power | Ensure the relay VCC is connected to the board's 5V pin, not 3V3. The mechanical electromagnet requires 5V to overcome spring tension |
| The LED on the relay module turns ON, but no click is heard | Internal relay contact fault | If the module indicator light toggles but there is no mechanical sound, the relay coil may be damaged or the power supply current is too low |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [09 - LED SOS Signal generator](09-led-sos-signal-generator.md)
- [11 - 5V DC Fan switch (via Relay/Transistor)](11-5v-dc-fan-switch.md) (Next project)
