# 19 - ESP32 Double Button AND Logic Gate

Use two push buttons to implement a hardware AND logic gate: the LED only turns on when **both** buttons are pressed simultaneously.

## Goal
Learn how to combine multiple digital inputs with a logical AND condition and understand how digital logic gates map to real circuit behaviour.

## What You Will Build
Two push buttons on GPIO 4 and GPIO 5 (each with a 10 kΩ pull-down resistor). An LED on GPIO 15 lights only when both buttons are held at the same time.

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
| Button A | Pin 1 (supply side) | 3V3 | Red | Connects 3V3 when pressed |
| Button A | Pin 2 (signal side) | GPIO4 | Yellow | Input A |
| 10 kΩ Resistor A | Leg 1 | GPIO4 | White | Pull-down for Button A |
| 10 kΩ Resistor A | Leg 2 | GND | Black | Ground reference |
| Button B | Pin 1 (supply side) | 3V3 | Red | Connects 3V3 when pressed |
| Button B | Pin 2 (signal side) | GPIO5 | Green | Input B |
| 10 kΩ Resistor B | Leg 1 | GPIO5 | White | Pull-down for Button B |
| 10 kΩ Resistor B | Leg 2 | GND | Black | Ground reference |
| LED | Anode (+) | GPIO15 via 330 Ω | Orange | AND gate output |
| LED | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** Both buttons use external pull-down resistors so each pin reads LOW at rest and HIGH when pressed. Place each 10 kΩ resistor directly from its signal pin to the GND rail — do not share a single resistor between both buttons.

## Code
```cpp
// Double Button AND Logic Gate
const int BTN_A   = 4;    // Input A
const int BTN_B   = 5;    // Input B
const int LED_PIN = 15;   // AND output

void setup() {
  pinMode(BTN_A,   INPUT);   // External pull-down fitted
  pinMode(BTN_B,   INPUT);   // External pull-down fitted
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Serial.begin(115200);
  Serial.println("AND Logic Gate ready.");
}

void loop() {
  bool a = digitalRead(BTN_A);   // true = pressed
  bool b = digitalRead(BTN_B);   // true = pressed

  // AND gate: output is HIGH only if BOTH inputs are HIGH
  bool out = a && b;
  digitalWrite(LED_PIN, out);

  Serial.print("A="); Serial.print(a);
  Serial.print("  B="); Serial.print(b);
  Serial.print("  OUT="); Serial.println(out);

  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, two **Button** components, and one **LED** onto the canvas.
2. Connect Button A to **GPIO4**, Button B to **GPIO5**, LED to **GPIO15**.
3. Paste the C++ code and click **Run**.
4. Press individual buttons and both together to see the AND truth table play out.

## Expected Output
Serial Monitor:
```
AND Logic Gate ready.
A=0  B=0  OUT=0
A=1  B=0  OUT=0
A=0  B=1  OUT=0
A=1  B=1  OUT=1
```

## Expected Canvas Behavior
* LED remains off when only one button is pressed.
* LED lights up only when both buttons are pressed simultaneously.
* Serial Monitor displays the three-column truth table in real time.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `bool a = digitalRead(BTN_A)` | Reads Button A — `true` (1) if HIGH, `false` (0) if LOW. |
| `bool b = digitalRead(BTN_B)` | Reads Button B in the same way. |
| `bool out = a && b` | C++ logical AND: `out` is `true` only when both `a` and `b` are `true`. |
| `digitalWrite(LED_PIN, out)` | Drives the LED HIGH or LOW to reflect the AND result. |

## Hardware & Safety Concept: Logic Gates in Software
A logic gate is a circuit that computes a Boolean function on one or more binary inputs to produce a single binary output. Traditionally, gates were built from transistors; microcontrollers let you implement identical behaviour in software with simple Boolean operators. The AND gate (`&&`) requires ALL inputs to be true for the output to be true. This principle is used in safety interlocks — for example, a machine that must confirm the guard door is closed AND the operator has pressed start before running.

## Try This! (Challenges)
1. **Truth table printer**: Print all four combinations (00, 01, 10, 11) and their AND outputs once at startup.
2. **Third button**: Add Button C on GPIO 18 and make a 3-input AND gate (`a && b && c`).
3. **LED blink on AND**: Instead of steady ON, blink the LED rapidly when the AND condition is met.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED turns on with only one button | Pull-down missing on the other button pin | Add a 10 kΩ resistor from the floating pin to GND |
| LED never turns on even with both buttons | Wrong GPIO numbers | Check pin assignments match your physical wiring |
| Inconsistent readings | Shared pull-down resistor | Each button signal pin needs its own pull-down |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [18 - ESP32 Pull-down Button Logic](18-esp32-pull-down-button-logic.md)
- [20 - ESP32 Double Button OR Logic Gate](20-esp32-double-button-or-logic.md)
- [26 - ESP32 Button Press Counter](26-esp32-button-press-counter.md)
