# 20 - ESP32 Double Button OR Logic Gate

Use two push buttons to implement an OR logic gate: the LED turns on when **either** button (or both) is pressed.

## Goal
Learn how to combine multiple digital inputs with a logical OR condition and contrast the behaviour with the AND gate from Project 19.

## What You Will Build
Two push buttons on GPIO 4 and GPIO 5 (each with a 10 kΩ pull-down resistor). An LED on GPIO 15 lights when at least one button is pressed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Tactile Push Button (×2) | `button` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 10 kΩ Resistor (×2) | `resistor` | No | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Button A | Pin 1 (supply side) | 3V3 | Red | Supply voltage |
| Button A | Pin 2 (signal side) | GPIO4 | Yellow | Input A |
| 10 kΩ Resistor A | Leg 1 | GPIO4 | White | Pull-down for Button A |
| 10 kΩ Resistor A | Leg 2 | GND | Black | Ground reference |
| Button B | Pin 1 (supply side) | 3V3 | Red | Supply voltage |
| Button B | Pin 2 (signal side) | GPIO5 | Green | Input B |
| 10 kΩ Resistor B | Leg 1 | GPIO5 | White | Pull-down for Button B |
| 10 kΩ Resistor B | Leg 2 | GND | Black | Ground reference |
| LED | Anode (+) | GPIO15 via 330 Ω | Orange | OR gate output |
| LED | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** The wiring is identical to Project 19. Only the firmware logic changes — `||` replaces `&&`. This clearly illustrates that hardware wiring and firmware logic are independent: the same circuit can implement different gates by changing one character in the code.

## Code
```cpp
// Double Button OR Logic Gate
const int BTN_A   = 4;    // Input A
const int BTN_B   = 5;    // Input B
const int LED_PIN = 15;   // OR output

void setup() {
  pinMode(BTN_A,   INPUT);   // External pull-down fitted
  pinMode(BTN_B,   INPUT);   // External pull-down fitted
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Serial.begin(115200);
  Serial.println("OR Logic Gate ready.");
}

void loop() {
  bool a = digitalRead(BTN_A);   // true = pressed
  bool b = digitalRead(BTN_B);   // true = pressed

  // OR gate: output is HIGH if EITHER input is HIGH
  bool out = a || b;
  digitalWrite(LED_PIN, out);

  Serial.print("A="); Serial.print(a);
  Serial.print("  B="); Serial.print(b);
  Serial.print("  OUT="); Serial.println(out);

  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, two **Button** components, and one **LED** onto the canvas.
2. Connect Button A to **GPIO4**, Button B to **GPIO5**, LED anode to **GPIO15**.
3. Paste the C++ code and click **Run**.
4. Press individual buttons and both together to observe OR behaviour.

## Expected Output
Serial Monitor:
```
OR Logic Gate ready.
A=0  B=0  OUT=0
A=1  B=0  OUT=1
A=0  B=1  OUT=1
A=1  B=1  OUT=1
```

## Expected Canvas Behavior
* LED lights up when Button A alone is pressed.
* LED lights up when Button B alone is pressed.
* LED lights up when both are pressed simultaneously.
* LED turns off only when neither button is pressed.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `bool out = a \|\| b` | C++ logical OR: `out` is `true` when at least one of `a` or `b` is `true`. |
| `digitalWrite(LED_PIN, out)` | Drives the LED to reflect the OR result. |
| `Serial.print` chain | Builds a formatted truth-table row on the Serial Monitor. |

## Hardware & Safety Concept: OR Gates in Real Systems
An OR gate outputs HIGH if any one of its inputs is HIGH. This models **any-source activation** — a common requirement in safety and automation. For example, a building alarm may sound if the front door sensor OR the window sensor OR the motion detector triggers. Software OR gates let a single microcontroller combine many event sources without extra hardware chips. Compare the truth table output from this project with Project 19 (AND): note that OR has three output-HIGH rows versus AND's single output-HIGH row.

## Try This! (Challenges)
1. **XOR gate**: Implement an exclusive-OR — LED on when inputs differ (`a != b`).
2. **NOR gate**: Invert the OR output (`!out`) so the LED is ON when both buttons are released.
3. **Three-input OR**: Add a third button on GPIO 18 and use `a || b || c`.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays on when no button pressed | Missing pull-down resistor | Add 10 kΩ from each signal pin to GND |
| Only one button works | Loose wire on second button's signal leg | Re-seat the GPIO5 wire into the breadboard |
| LED very dim | Wrong current-limiting resistor | Verify a 330 Ω resistor is on the LED anode |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [19 - ESP32 Double Button AND Logic Gate](19-esp32-double-button-and-logic.md)
- [21 - ESP32 TTP223 Touch Lamp](21-esp32-ttp223-touch-lamp.md)
- [26 - ESP32 Button Press Counter](26-esp32-button-press-counter.md)
