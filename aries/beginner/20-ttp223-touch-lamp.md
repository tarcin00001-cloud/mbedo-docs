# 20 - TTP223 Touch Lamp

Build a touch-sensitive lamp using a TTP223 capacitive touch sensor module and an external LED on the VEGA ARIES v3 board.

## Goal
Learn how to interface active digital sensor modules (like the TTP223 capacitive touch sensor), understand active-high output signals, and control an output device based on sensor input.

## What You Will Build
A TTP223 capacitive touch sensor is powered by 3.3V and connected to `GPIO 17`. An external LED is connected to `GPIO 15`. When a finger touches the sensor's metal pad, the module outputs a digital HIGH signal, which causes the ARIES board to turn on the LED. When the finger is removed, the module outputs LOW, and the LED turns off.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| TTP223 Touch Sensor | `ttp223` (or generic digital input) | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| TTP223 Touch Sensor | VCC | 3.3V | Red | Power connection |
| TTP223 Touch Sensor | GND | GND | Black | Ground connection |
| TTP223 Touch Sensor | OUT | GPIO 17 | Green | Digital output signal |
| LED | Anode (+) | GPIO 15 | Red | Digital control signal (through resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The TTP223 is an active component and requires 3.3V power to operate. Connect its VCC pin to 3.3V and GND pin to GND. Do not connect OUT directly to power or ground.

## Code
```cpp
// TTP223 Touch Lamp - VEGA ARIES v3
const int TOUCH_PIN = 17;
const int LED_PIN = 15;

void setup() {
  // Configure the touch sensor pin as input
  pinMode(TOUCH_PIN, INPUT);
  
  // Configure the LED pin as output
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  // Read the touch sensor state
  int touchState = digitalRead(TOUCH_PIN);
  
  // If the sensor is touched, the module outputs HIGH
  if (touchState == HIGH) {
    digitalWrite(LED_PIN, HIGH);
  } else {
    digitalWrite(LED_PIN, LOW);
  }
  
  delay(10); // Small delay to stabilize reading loop
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **TTP223 Touch Sensor** widget, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire the TTP223's **VCC** to **3.3V** and **GND** to **GND**.
3. Wire the TTP223's **OUT** pin to **GPIO 17**.
4. Wire one end of the Resistor to **GPIO 15** and the other to the LED's anode (+).
5. Wire the LED's cathode (-) to **GND**.
6. Paste the code into the editor.
7. Click **Run**.
8. Click and hold the TTP223 sensor widget on the canvas to see the LED light up. Release to turn it off.

## Expected Output
Serial Monitor:
```
System Initialized.
Touch Sensor: HIGH | LED: ON
Touch Sensor: LOW  | LED: OFF
```

## Expected Canvas Behavior
* The external LED turns ON and glows bright red as long as the TTP223 Touch Sensor is clicked/held down.
* The LED goes dark when the sensor widget is released.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(TOUCH_PIN, INPUT)` | Sets GPIO 17 as a digital input pin to read signals sent by the TTP223. |
| `digitalRead(TOUCH_PIN)` | Samples the state of the TTP223 module output. |
| `touchState == HIGH` | Condition checking if touch contact is detected (HIGH logic levels). |

## Hardware & Safety Concept
* **Capacitive Touch Sensing**: Capacitive touch sensors detect changes in electrical capacitance. The human body has a certain capacitance, and when a finger gets close to the sensor pad, it changes the capacitance of the onboard capacitor. The TTP223 IC detects this minute change, filters it, and outputs a clean digital HIGH/LOW signal.
* **Module Power Isolation**: Because active sensors run off the 3.3V power bus, it is important to ensure they do not exceed their voltage ratings. The TTP223 operates safely from 2.0V to 5.5V, making the ARIES board's 3.3V rail ideal.

## Try This! (Challenges)
1. **Latching Switch**: Modify the code using variables (similar to Project 21) so that touching the sensor once turns the LED ON, and touching it again turns the LED OFF.
2. **Double Touch Speed Test**: Make the LED blink rapidly only if the sensor remains touched.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays ON continuously | Calibration error or short circuit | Make sure nothing metallic is touching the sensor's pad; power-cycle the board to recalibrate the sensor |
| Sensor does not respond | VCC/GND reversed | Check the wiring of the TTP223 power lines immediately to avoid damaging the sensor |
| LED is dim | Resistor value too high | Ensure the LED resistor is 220 Ω, not a larger value |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [19 - Double Button OR Gate](19-double-button-or-gate.md)
- [21 - Touch Sensor Toggle switch](21-touch-sensor-toggle-switch.md) (Next project)
