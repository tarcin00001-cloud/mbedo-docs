# 11 - 5V DC Fan switch (via Relay/Transistor)

Control a 5V DC cooling fan using a digital output pin on the VEGA ARIES v3 board through a transistor or relay driver.

## Goal
Learn how to control DC motors and fans using digital outputs, understand the configuration of high-current transistor switches, and implement basic warning indicators.

## What You Will Build
A 5V DC cooling fan is connected to the VEGA ARIES v3 board through a transistor driver circuit controlled by pin `GPIO 14`. The fan is toggled ON for 5 seconds and OFF for 5 seconds. A warning LED on `GPIO 15` lights up when the fan is running.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 5V DC Fan | `fan` | Yes | Yes |
| NPN Transistor (PN2222) | `transistor` | Yes | Yes |
| 1N4007 Diode (Flyback) | `diode` | Yes | Yes |
| LED & 220 Ω Resistor | `led` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Transistor | Base | GPIO 14 | Yellow | Base signal (through 1 kΩ resistor) |
| Transistor | Collector | Fan (-) | Black | Connects to fan negative terminal |
| Transistor | Emitter | GND | Black | Ground connection |
| DC Fan | Positive (+) | 5V | Red | Fan power supply (5V) |
| Flyback Diode | Cathode / Anode | Fan (+) / Fan (-) | Red / Black | Clamping diode across fan terminals |
| Warning LED | Anode (+) | GPIO 15 | Orange | Warning indicator (via 220 Ω resistor) |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** DC motors and fans generate electrical noise when starting. Always connect a 1N4007 flyback diode across the fan terminals (cathode to positive, anode to negative) to clamp inductive voltage spikes.

## Code
```cpp
// 5V DC Fan Switch - VEGA ARIES v3
const int FAN_PIN = 14;
const int WARNING_LED = 15;

void setup() {
  // Configure both pins as digital outputs
  pinMode(FAN_PIN, OUTPUT);
  pinMode(WARNING_LED, OUTPUT);
}

void loop() {
  // Activate fan and turn warning LED ON
  digitalWrite(FAN_PIN, HIGH);
  digitalWrite(WARNING_LED, HIGH);
  delay(5000);
  
  // Deactivate fan and turn warning LED OFF
  digitalWrite(FAN_PIN, LOW);
  digitalWrite(WARNING_LED, LOW);
  delay(5000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **DC Fan**, **NPN Transistor**, **LED**, and **Resistors** onto the canvas.
2. Wire the Transistor Base to **GPIO 14** (through a 1 kΩ resistor), Emitter to **GND**, and Collector to the Fan's negative lead.
3. Connect the Fan's positive lead to **5V**.
4. Wire the LED's anode to **GPIO 15** (through a 220 Ω resistor) and the cathode to **GND**.
5. Paste the code into the editor.
6. Click **Run**.
7. Observe the DC Fan spinning on the canvas for 5 seconds, followed by 5 seconds of rest.

## Expected Output
Serial Monitor:
```
System Initialized.
Cooling Cycle Started:
  Fan State: ON  | Warning LED: ON
  Fan State: OFF | Warning LED: OFF
```

## Expected Canvas Behavior
* The Fan widget spins rapidly and the warning LED lights up for 5 seconds, then both stop for 5 seconds before repeating.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(FAN_PIN, OUTPUT)` | Sets GPIO 14 to digital output mode to control the transistor gate. |
| `digitalWrite(FAN_PIN, HIGH)` | Applies voltage to the transistor base, switching it ON and grounding the fan to complete the circuit. |
| `delay(5000)` | Keeps the fan and LED active for 5 seconds. |

## Hardware & Safety Concept: Inductive Loads and Transistor Switching
* **Inductive Loads**: DC motors and fans are inductive loads. When power is disconnected, the magnetic field in the motor windings collapses, causing a reverse voltage spike (up to 100V). Without a flyback diode, this voltage can burn out the switching transistor.
* **Transistor Switching**: Microcontrollers cannot directly power motors because GPIO pins can only supply a few milliamps (8–12 mA), while even a small fan requires 100–200 mA. A transistor acts as an amplifier, allowing a tiny pin current to control a much larger current from the 5V rail.

## Try This! (Challenges)
1. **Pulse Cooling**: Modify the cycle to run the fan for 10 seconds and turn it OFF for 2 seconds.
2. **Double Alert**: Modify the warning LED to flash rapidly three times before the fan turns ON to warn users of moving blades.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Fan does not spin but LED is ON | Transistor wired backwards | Verify the transistor pinout (Emitter, Base, Collector). Swap collector and emitter connections if wired incorrectly |
| The board resets when the fan starts | Voltage sag / high current | The fan is drawing too much current from the USB rail. Power the fan using an external 5V source, keeping grounds connected |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [10 - Relay Module Switch](10-relay-module-switch.md)
- [12 - Dual RGB LED Sync](12-dual-rgb-led-sync.md) (Next project)
