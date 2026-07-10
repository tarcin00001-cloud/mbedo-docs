# 46 - ESP32 Potentiometer Servo Control

Use a potentiometer to directly control the angle of a servo motor — the knob position maps to the servo shaft angle in real time.

## Goal
Learn how to use the ESP32Servo library to control a servo motor and how to map an ADC reading from a potentiometer to a servo angle using `map()`.

## What You Will Build
A potentiometer wiper on GPIO 34. The ADC reading (0–4095) is mapped to a servo angle (0–180°). A servo motor on GPIO 5 moves its shaft to match the knob position in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Left leg (VCC end) | 3V3 | Red | Reference voltage |
| Potentiometer | Right leg (GND end) | GND | Black | Ground reference |
| Potentiometer | Wiper (middle leg) | GPIO34 | Yellow | ADC control input |
| Servo Motor | Signal (orange/yellow) | GPIO5 | Orange | PWM servo control |
| Servo Motor | VCC (red) | 5V (Vin) | Red | Servo power — needs 5 V |
| Servo Motor | GND (brown/black) | GND | Black | Ground return |

> **Wiring tip:** Power the servo from the ESP32's Vin pin (5 V from USB) or an external 5 V supply — do NOT power it from 3V3. An SG90 servo draws up to 500 mA on stall; if powered from the ESP32's 3V3 regulator the voltage will droop and the ESP32 may reset. The servo signal wire (GPIO 5) operates at 3.3 V logic, which is compatible with all standard servos.

## Code
```cpp
// Potentiometer Servo Control using ESP32Servo library
#include <ESP32Servo.h>

const int POT_PIN   = 34;
const int SERVO_PIN = 5;

Servo myServo;

void setup() {
  myServo.attach(SERVO_PIN);   // Attach servo to GPIO 5
  Serial.begin(115200);
  Serial.println("Potentiometer Servo Control ready.");
}

void loop() {
  int raw   = analogRead(POT_PIN);              // 0–4095
  int angle = map(raw, 0, 4095, 0, 180);        // Map to 0–180°
  angle     = constrain(angle, 0, 180);          // Safety clamp

  myServo.write(angle);

  Serial.print("ADC: "); Serial.print(raw);
  Serial.print("  |  Angle: "); Serial.print(angle);
  Serial.println("°");

  delay(20);   // ~50 Hz — smooth servo response
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer**, and **Servo** onto the canvas.
2. Connect Potentiometer **wiper** to **GPIO34**, Servo **signal** to **GPIO5**.
3. Paste the code and click **Run**.
4. Drag the potentiometer slider — the servo widget rotates to match.

## Expected Output
Serial Monitor:
```
Potentiometer Servo Control ready.
ADC: 0     |  Angle: 0°
ADC: 1024  |  Angle: 45°
ADC: 2048  |  Angle: 90°
ADC: 3072  |  Angle: 135°
ADC: 4095  |  Angle: 180°
```

## Expected Canvas Behavior
* Servo widget arm points to 0° when the slider is fully left.
* Servo widget arm points to 180° when the slider is fully right.
* Intermediate positions track the slider smoothly.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `#include <ESP32Servo.h>` | Includes the ESP32Servo library which handles LEDC-based servo PWM internally. |
| `myServo.attach(SERVO_PIN)` | Configures GPIO 5 to generate 50 Hz servo PWM pulses. |
| `map(raw, 0, 4095, 0, 180)` | Linearly converts the 12-bit ADC range to 0–180°. |
| `myServo.write(angle)` | Commands the servo to move to the specified angle. |
| `delay(20)` | 50 Hz update rate — matches the servo's standard 50 Hz PWM refresh. |

## Hardware & Safety Concept: RC Servo PWM Protocol
A standard hobby servo expects a **50 Hz PWM signal** with a pulse width between 1000 µs (0°) and 2000 µs (180°). The ESP32Servo library uses the LEDC hardware PWM peripheral to generate these precise pulses. The servo motor contains a small DC motor, a gear reduction box, and a potentiometer feedback loop — a complete closed-loop position controller in one package. When you command 90°, the servo's internal controller drives its motor until its internal pot reads the voltage corresponding to 90°, then holds. Servo motors are used in: RC aircraft control surfaces, robotic arms, camera gimbals, throttle actuators, and valve controllers.

## Try This! (Challenges)
1. **Sweep limits**: Constrain the servo to a 45°–135° range by changing the map output bounds.
2. **Speed limit**: Instead of writing directly, move the servo 1° per loop iteration toward the target to create a slew-rate-limited smooth sweep.
3. **Two servos**: Add a second servo on GPIO 15 and a second potentiometer on GPIO 35 for a two-axis pan-tilt system.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo jitters constantly | Power supply noise from USB | Add a 100 µF electrolytic capacitor across servo VCC and GND |
| Servo only moves to 0° or 180° | Potentiometer wiper not connected | Verify GPIO 34 wire is in the pot wiper (middle) pin |
| ESP32 resets when servo moves | Servo drawing too much current from 3V3 | Move servo VCC to 5 V (Vin) or an external supply |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [31 - ESP32 Potentiometer ADC Read](31-esp32-potentiometer-adc-read.md)
- [32 - ESP32 Potentiometer LED Dimmer](32-esp32-potentiometer-led-dimmer.md)
- [47 - ESP32 Analog Joystick X-Axis Logging](47-esp32-analog-joystick-x-axis-logging.md)
