# 21 - Touch Sensor Toggle switch

Convert a capacitive touch sensor into a latching toggle switch to turn an external LED ON and OFF on the VEGA ARIES v3 board.

## Goal
Learn how to implement latching behavior in software by detecting edge transitions (rising edge) on a sensor module and using state variables to remember output state.

## What You Will Build
A TTP223 capacitive touch sensor is connected to `GPIO 17`. An external LED is connected to `GPIO 15`. Instead of the LED lighting up only while the sensor is touched, this project creates a "latching" behavior: touch the sensor once to turn the LED ON, and touch it again to turn the LED OFF.

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

> **Wiring tip:** Connect the TTP223 module VCC and GND pins to the 3.3V and GND rails. Since it operates in active-high mode by default, its OUT pin will sit at 0V (LOW) when idle and jump to 3.3V (HIGH) on touch.

## Code
```cpp
// Touch Sensor Toggle Switch - VEGA ARIES v3
const int TOUCH_PIN = 17;
const int LED_PIN = 15;

int lastTouchState = LOW; // Remember previous sensor state
int ledState = LOW;       // Track current LED state (latching state)

void setup() {
  // Configure the touch sensor pin as input
  pinMode(TOUCH_PIN, INPUT);
  
  // Configure the LED pin as output
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, ledState);
}

void loop() {
  int currentTouchState = digitalRead(TOUCH_PIN);
  
  // Detect a rising edge (transition from LOW to HIGH)
  if (lastTouchState == LOW && currentTouchState == HIGH) {
    ledState = !ledState;            // Toggle the state
    digitalWrite(LED_PIN, ledState); // Update LED output
    delay(200);                      // Debounce cooldown to prevent double toggling
  }
  
  lastTouchState = currentTouchState; // Save state for next iteration
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **TTP223 Touch Sensor** widget, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire the TTP223's **VCC** to **3.3V**, **GND** to **GND**, and **OUT** to **GPIO 17**.
3. Wire the **220 Ω Resistor** in series between **GPIO 15** and the LED's anode (+). Connect the LED's cathode (-) to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Tap (click and release) the TTP223 sensor widget. The LED turns ON and stays ON.
7. Tap the TTP223 sensor widget again. The LED turns OFF and stays OFF.

## Expected Output
Serial Monitor:
```
System Initialized.
Touch Detected! Toggling LED state.
GPIO 15 State: HIGH
Touch Detected! Toggling LED state.
GPIO 15 State: LOW
```

## Expected Canvas Behavior
* Tapping the touch sensor toggles the external LED's illumination status.
* The LED retains its state permanently until another touch is registered.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `int lastTouchState = LOW` | State variable to store the previous sensor reading to detect edges. |
| `lastTouchState == LOW && currentTouchState == HIGH` | Condition logic checking for a rising edge transition. |
| `ledState = !ledState` | Toggles the logic state of the LED state variable using the logical NOT (`!`) operator. |
| `delay(200)` | Cooldown timer ensuring multiple false rising edges are not detected during a single physical touch. |

## Hardware & Safety Concept
* **Edge Detection vs Level Detection**: Level detection triggers actions based on a continuous state (e.g. while button is LOW). Edge detection triggers an action once, only at the exact instant the state changes. This project uses rising edge detection to change states exactly once per touch.
* **Interpretation Debounce Limits**: In interpreted platforms, executing code loops takes a fraction of a millisecond. A cooldown delay of 200 ms is perfect for touch sensors as touch events typically last between 100 ms and 300 ms.

## Try This! (Challenges)
1. **Flash State on Tap**: Modify the code so that when toggled ON, the LED blinks repeatedly, and when toggled OFF, it remains dark.
2. **Onboard LED Sync**: Synchronize the onboard RGB Blue LED (`LED_B`) to light up in opposite states of the external LED.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED toggles twice rapidly on one touch | Cooldown delay too short | Increase the `delay(200)` to `delay(300)` or `delay(400)` |
| LED does not toggle | Faulty wiring or wrong pin | Check that the TTP223 OUT pin is wired to GPIO 17, and the LED is wired to GPIO 15 |
| LED flickers when hand gets close without touching | High sensor sensitivity | Capacitive sensors can trigger from proximity. Ensure the board and sensor are on a clean, non-conductive surface |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [20 - TTP223 Touch Lamp](20-ttp223-touch-lamp.md)
- [22 - Limit Switch indicator](22-limit-switch-indicator.md) (Next project)
