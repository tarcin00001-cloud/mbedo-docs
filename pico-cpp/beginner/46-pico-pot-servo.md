# 46 - Pico Pot Servo

Control a servo motor's angle dynamically using a potentiometer.

## Goal
Learn how to map analog ADC input ranges (0-4095) to servo angle limits (0-180) and write to servo actuators.

## What You Will Build
A manual servo pointer:
- **Potentiometer (GP26)**: Reads 0 to 4095.
- **Servo Motor (GP10)**: Moves its output arm dynamically to match the potentiometer position (from 0 degrees to 180 degrees).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Potentiometer | Pin 1 (VCC) | 3.3V | Power supply |
| Potentiometer | Pin 2 (Wiper) | GP26 | Analog input |
| Potentiometer | Pin 3 (GND) | GND | Ground return |
| Servo Motor | Signal (Orange) | GP10 | PWM control pin |
| Servo Motor | Power (Red) | 5V (or 3V3) | Power supply |
| Servo Motor | Ground (Brown) | GND | Ground return |

## Code
```cpp
#include <Servo.h>

const int POT_PIN   = 26;
const int SERVO_PIN = 10;

Servo myServo;

void setup() {
  pinMode(POT_PIN, INPUT);
  myServo.attach(SERVO_PIN);
}

void loop() {
  int rawValue = analogRead(POT_PIN); // Reads 0 to 4095

  // Map 12-bit ADC (0-4095) to Servo Degrees (0-180)
  int angle = rawValue * 180 / 4095;

  myServo.write(angle); // Update servo shaft position

  delay(50); // Small delay to let servo reach target position
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, and **Servo Motor** onto the canvas.
2. Connect Potentiometer: **Pin 1** to **3V3**, **Pin 2** to **GP26**, **Pin 3** to **GND**.
3. Connect Servo: **Signal** to **GP10**, **Power** to **5V** (or **VBUS**), **Ground** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Drag the potentiometer slider on canvas and watch the servo sweep.

## Expected Output

Terminal:
```
Simulation active. Mapping GP26 to Servo GP10.
```

## Expected Canvas Behavior
| Potentiometer Position | GP26 Input Value | Servo Output Angle | Servo Pointer Position |
| --- | --- | --- | --- |
| Far Left | 0 | 0 | 0° (Fully Left) |
| Center | 2048 | 90 | 90° (Center) |
| Far Right | 4095 | 180 | 180° (Fully Right) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `myServo.attach(SERVO_PIN)` | Configures GP10 to output a standard 50 Hz PWM signal required by hobby servos. |
| `myServo.write(angle)` | Adjusts the pulse width of the PWM signal to command the internal servo controller to target a specific angle. |

## Hardware & Safety Concept: Servo Power Draw
Hobby servo motors contain a DC motor, gears, and a feedback potentiometer. When the servo moves or holds a heavy load, it can draw high current spikes (500mA to 2A). Powering a servo directly from the Pico's 3.3V rail can trigger a brownout reset on the RP2040 chip. Always connect the servo power line to the 5V USB line (VBUS) or an external power supply with a shared common ground.

## Try This! (Challenges)
1. **Sweeping Indicator**: Add a red LED on GP15 that flashes rapidly only if the servo is near its limit angles (e.g. `< 10` or `> 170`).
2. **Reverse Servo**: Modify the map calculation so the servo sweeps in the opposite direction of the potentiometer dial.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo vibrates or moves erratically | Shared ground missing | Ensure the servo ground line is connected directly to one of the Pico's GND pins to establish a common reference voltage. |

## Mode Notes
This basic servo control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [31 - Pico Potentiometer ADC](31-pico-potentiometer-adc.md)
- [51 - Pico Servo Sweep](../../intermediate/51-pico-servo-sweep.md)
