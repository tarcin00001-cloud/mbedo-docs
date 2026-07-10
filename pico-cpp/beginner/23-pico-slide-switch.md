# 23 - Pico Slide Switch

Log and monitor the state of a slide switch using the Raspberry Pi Pico.

## Goal
Learn how to read SPDT (Single Pole Double Throw) switch inputs and display their current states on the Serial Monitor.

## What You Will Build
A switch state monitor:
- **Slide Switch (GP16)**: Reads `HIGH` when slid to the VCC terminal, and `LOW` when slid to the GND terminal.
- **Warning LED (GP15)**: Automatically matches the position of the slide switch.
- **Serial Monitor**: Prints the switch state ("Switch: ON" or "Switch: OFF") every 500 ms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Slide Switch (SPDT) | `button` | Yes (or switch module) | Yes |
| LED (Red) | `led` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (pull-down) |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Slide Switch | Pin 1 (VCC) | 3.3V | Power supply connection |
| Slide Switch | Pin 2 (Common) | GP16 | Digital input (needs 10k pull-down to GND) |
| Slide Switch | Pin 3 (GND) | GND | Ground connection |
| Red LED | Anode | GP15 | Switch state indicator |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int SWITCH_PIN = 16;
const int LED_PIN    = 15;

void setup() {
  Serial.begin(9600);
  pinMode(SWITCH_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  Serial.println("Slide Switch Monitor Active");
}

void loop() {
  int switchState = digitalRead(SWITCH_PIN);

  if (switchState == HIGH) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("Switch: ON");
  } else {
    digitalWrite(LED_PIN, LOW);
    Serial.println("Switch: OFF");
  }

  delay(500); // Sample twice per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Slide Switch** (represented by button/switch), and **Red LED** onto the canvas.
2. Connect Switch: **Pin 1** to **3.3V**, **Pin 2** to **GP16** (with 10k resistor to GND), **Pin 3** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Toggle the slide switch and observe the LED and Serial console outputs.

## Expected Output

Terminal:
```
Slide Switch Monitor Active
Switch: OFF
Switch: ON
```

## Expected Canvas Behavior
| Switch Position | GP16 Input State | LED state (GP15) | Serial Output |
| --- | --- | --- | --- |
| Ground Side | LOW | LOW | `Switch: OFF` |
| Power Side | HIGH | HIGH | `Switch: ON` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial.begin(9600)` | Opens the hardware UART serial port at a baud rate of 9600 bits per second for printing log messages. |
| `Serial.println("Switch: ON")` | Transmits an ASCII text string followed by a newline code over USB to the terminal console. |

## Hardware & Safety Concept: SPDT Switch Architecture
SPDT switches route a common center pin (Pin 2) to either pin 1 or pin 3 depending on mechanical slider position. Because pin 2 is actively connected to either 3.3V or GND, it avoids floating inputs without requiring external resistors if both power and ground pins are connected. However, adding a pull-down resistor protects the pin during transitions.

## Try This! (Challenges)
1. **Change Baud Rate**: Modify the code and simulator configuration to run at 115200 baud rate.
2. **Alert Tone**: Sound a buzzer briefly on GP14 only when the switch transitions from OFF to ON.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Serial Monitor displays blank lines | Serial port not initialized | Confirm that `Serial.begin(9600)` is included in your `setup()` block. |

## Mode Notes
This basic GPIO and Serial logging project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [15 - Analog Meter Serial](../../beginner/15-analog-meter-serial.md) (Uno equivalent)
- [16 - Pico Button LED](16-pico-button-led.md)
