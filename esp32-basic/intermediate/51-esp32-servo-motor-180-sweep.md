# 51 - ESP32 Servo Motor 180° Sweep

Drive a servo motor through a full 0°–180°–0° sweep cycle using the ESP32Servo library and the LEDC hardware PWM peripheral.

## Goal
Learn how to set up the ESP32Servo library, attach a servo to a GPIO pin, and command a smooth continuous sweep between 0° and 180° in equal degree steps.

## What You Will Build
A servo motor on GPIO 5. The code sweeps the shaft from 0° to 180° in 1° steps, pauses at each end for 500 ms, then reverses — creating a continuous back-and-forth pendulum motion.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Servo Motor | Signal (orange/yellow) | GPIO5 | Orange | PWM servo control signal |
| Servo Motor | VCC (red) | 5V (Vin) | Red | Servo power — must be 5 V |
| Servo Motor | GND (brown/black) | GND | Black | Ground return |

> **Wiring tip:** The SG90 servo can draw up to 500 mA at stall. Power it from the ESP32's **Vin** pin (5 V from USB), never from 3V3. If the ESP32 resets during servo movement, the 3V3 regulator is overloaded — add an external 5 V supply. The signal wire operates at 3.3 V logic and is fully compatible with the SG90.

## Code
```cpp
// Servo Motor 180° Sweep using ESP32Servo library
#include <ESP32Servo.h>

const int SERVO_PIN = 5;
Servo myServo;

void setup() {
  myServo.attach(SERVO_PIN);
  Serial.begin(115200);
  Serial.println("Servo 180° Sweep starting.");
}

void loop() {
  // Sweep from 0° to 180°
  Serial.println("Sweeping 0° → 180°");
  for (int angle = 0; angle <= 180; angle++) {
    myServo.write(angle);
    delay(10);   // 10 ms per degree = 1.8 s total sweep
  }
  delay(500);    // Pause at 180°

  // Sweep from 180° back to 0°
  Serial.println("Sweeping 180° → 0°");
  for (int angle = 180; angle >= 0; angle--) {
    myServo.write(angle);
    delay(10);
  }
  delay(500);    // Pause at 0°
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Servo** onto the canvas.
2. Connect Servo **signal** to **GPIO5**.
3. Paste the code and click **Run**.
4. Watch the servo widget arm sweep back and forth on the canvas.

## Expected Output
Serial Monitor:
```
Servo 180° Sweep starting.
Sweeping 0° → 180°
Sweeping 180° → 0°
Sweeping 0° → 180°
```

## Expected Canvas Behavior
* The servo widget arm moves steadily from the 0° position to the 180° position over ~1.8 seconds.
* It pauses briefly at each end before reversing.
* The cycle repeats continuously.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `#include <ESP32Servo.h>` | Imports the ESP32Servo library which wraps LEDC to produce standard 50 Hz servo pulses. |
| `myServo.attach(SERVO_PIN)` | Binds the servo object to GPIO 5 and begins PWM output. |
| `for (int angle = 0; angle <= 180; angle++)` | Steps through every integer degree from 0 to 180. |
| `myServo.write(angle)` | Commands the servo to move to the specified angle (1000–2000 µs pulse). |
| `delay(10)` | 10 ms per step — allows the servo time to mechanically move before the next command. |

## Hardware & Safety Concept: RC Servo PWM Control
Standard hobby servos respond to a 50 Hz PWM signal. The pulse width (not duty cycle) encodes the target angle: 1000 µs = 0°, 1500 µs = 90°, 2000 µs = 180°. The ESP32Servo library uses an LEDC timer channel to generate these precise pulse widths in hardware. Sending angle steps smaller than the servo's mechanical resolution (typically 1°) produces smooth analogue-looking motion instead of visible stepping. The servo's internal PID controller drives its motor until the internal position potentiometer matches the commanded angle — this closed-loop design gives consistent positioning regardless of load (within torque limits).

## Try This! (Challenges)
1. **Variable speed**: Replace `delay(10)` with a potentiometer-controlled delay (read GPIO 34, map to 5–50 ms) to vary sweep speed in real time.
2. **Partial sweep**: Modify the limits to sweep between 45° and 135° only.
3. **Push-button direction**: Add a button on GPIO 4 — each press reverses the sweep direction immediately.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo buzzes and vibrates at 0° or 180° | Servo hitting mechanical end-stops | Change limits to `angle = 5` and `angle = 175` |
| ESP32 resets mid-sweep | Servo drawing too much current | Move servo VCC to an external 5 V supply |
| Servo moves erratically | `delay(10)` too short for slow servo | Increase to `delay(15)` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - ESP32 Potentiometer Servo Control](../beginner/46-esp32-potentiometer-servo-control.md)
- [52 - ESP32 Dual Servo Coordinates Sweep](52-esp32-dual-servo-coordinates-sweep.md)
- [66 - ESP32 Servo Position Display](66-esp32-servo-position-display.md)
