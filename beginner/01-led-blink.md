# 01 - LED Blink

Make an LED flash on and off. Your first MbedO sketch.

## Goal
Write a sketch that turns an LED on for one second and off for one second, repeating
forever. Learn the fundamental structure of every Arduino program: `setup()` and `loop()`.

## What You Will Build
An LED connected to pin D13 blinks at a steady 1-second rate. D13 is also connected
to the onboard LED on the physical Arduino Uno board - both the canvas LED and the
real onboard LED would flash at the same time on hardware.

**Why D13?** Arduino Uno maps D13 to its onboard LED, making it the most reliable
first test pin - no external circuit needed to confirm the sketch is running.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| LED Pin | Arduino Pin | Notes |
| --- | --- | --- |
| A (Anode, longer leg) | D13 | Signal pin — this pin is toggled HIGH/LOW |
| C (Cathode, shorter leg) | GND | Ground reference |

> On real hardware, always place a **220 ohm resistor** in series between D13 and the
> LED anode to limit current. In MbedO simulation the resistor is optional.

## Code
```cpp
// Pin number for the LED
const int LED_PIN = 13;

void setup() {
  // Tell the Arduino that pin 13 will be used as an output
  pinMode(LED_PIN, OUTPUT);

  // Start the Serial Monitor connection at 9600 baud
  Serial.begin(9600);
  Serial.println("LED Blink ready");
}

void loop() {
  // Turn the LED ON (send 5V to pin 13)
  digitalWrite(LED_PIN, HIGH);
  delay(1000);             // Wait 1000 milliseconds (1 second)

  // Turn the LED OFF (send 0V to pin 13)
  digitalWrite(LED_PIN, LOW);
  delay(1000);             // Wait 1000 milliseconds (1 second)
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** onto the canvas.
2. Drag **LED** onto the canvas.
3. Connect LED **A** (Anode) to Arduino **D13**.
4. Connect LED **C** (Cathode) to Arduino **GND**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.

## Expected Output

Serial Monitor:
```
LED Blink ready
```

### Expected Canvas Behavior

| Time | Pin D13 | LED on Canvas |
| --- | --- | --- |
| 0 - 1 sec | HIGH (5V) | ON |
| 1 - 2 sec | LOW (0V) | OFF |
| 2 - 3 sec | HIGH (5V) | ON |
| Repeats | Alternates | Blinks |

The `LED Blink ready` message appears once in the Terminal. The LED then blinks
silently - no further Serial output is expected.

## Code Walkthrough

| Line | What It Does |
| --- | --- |
| `const int LED_PIN = 13;` | Declares a constant integer called `LED_PIN` with value `13`. Using a named constant means if you change the pin later, you only edit one line. |
| `void setup()` | Runs **once** when the Arduino powers on or resets. Use it to configure pins and start libraries. |
| `pinMode(LED_PIN, OUTPUT)` | Tells the Arduino that pin 13 is an **output** — it will push voltage out, not read it in. |
| `Serial.begin(9600)` | Opens the Serial Monitor communication channel at 9600 bits per second (baud). |
| `void loop()` | Runs **forever**, repeating from top to bottom continuously after `setup()` finishes. |
| `digitalWrite(LED_PIN, HIGH)` | Sends 5V to pin 13, completing the circuit and lighting the LED. |
| `delay(1000)` | Pauses the sketch for 1000 ms (1 second). Nothing else runs during this time. |
| `digitalWrite(LED_PIN, LOW)` | Brings pin 13 to 0V, breaking the circuit and turning the LED off. |

## Hardware & Safety Concept: LEDs and Current Limiting

An **LED** (Light-Emitting Diode) only allows current to flow in one direction - from
Anode (+) to Cathode (-). This is why polarity matters:

- **Anode (A, longer leg)** - connects to the signal pin (positive side)
- **Cathode (C, shorter leg)** - connects to GND (negative side)

Without a **current-limiting resistor**, the LED draws as much current as the pin can
supply, which can permanently damage both the LED and the Arduino output pin. The
Arduino Uno GPIO pins are rated at a maximum of **40 mA**. A typical red LED
needs only **10-20 mA** at around **2.0 V** forward voltage. Using Ohm's Law:

    R = (V_supply - V_LED) / I = (5V - 2V) / 0.015A = 200 ohm

So a **220 ohm** resistor (the nearest standard value) is the correct choice.

> On the MbedO canvas, current limiting is handled by the simulation — but always
> wire the resistor on real hardware.

## Try This! (Challenges)

1. **Change the Blink Speed**: Change both `delay(1000)` values to `delay(200)`.
   How fast does the LED blink? Now try `delay(50)`. At what speed does your eye
   stop seeing individual blinks and perceive the LED as continuously on?
   *(This is called the persistence of vision threshold - around 50 Hz.)*

2. **Heartbeat Pattern**: Make the LED mimic a heartbeat — two quick flashes
   followed by a long pause. Replace the contents of `loop()` with:
   ```cpp
   digitalWrite(LED_PIN, HIGH); delay(80);
   digitalWrite(LED_PIN, LOW);  delay(80);
   digitalWrite(LED_PIN, HIGH); delay(80);
   digitalWrite(LED_PIN, LOW);  delay(700);
   ```
   Can you adjust the timings to feel more like a real heartbeat at 60 BPM?

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED does not flash at all | Anode and cathode reversed | Swap wires: A to D13, C to GND |
| LED stays on permanently | Missing `digitalWrite(LED_PIN, LOW)` or second `delay` | Check the `loop()` function has both HIGH and LOW blocks |
| LED is very dim | Anode connected to a high-resistance path | Connect A directly to D13, not through an unintended resistor chain |
| No text in Terminal | Baud rate mismatch | Set the Terminal baud rate dropdown to **9600** |
| `LED Blink ready` prints, then nothing | Expected - Serial only prints once in `setup()` | The LED blinks silently after that; watch the canvas LED visually |

## Mode Notes
These patterns (`pinMode`, `digitalWrite`, `delay`, `Serial.begin`, `Serial.println`)
are supported by MbedO interpreted mode and are safe for this beginner project.
No external library is required.

> See [Simulation Modes](../../courses/simulation-modes.md) if you are unsure which
> runtime to select.

## Related Projects
- [02 - LED Fade](02-led-fade.md) - Control brightness with PWM
- [03 - Breathing LED](03-breathing-led.md) - Smooth pulsing effect
- [11 - Button ON/OFF](11-button-onoff.md) - Add user input to control the LED
