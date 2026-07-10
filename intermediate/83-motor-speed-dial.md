# 83 - Motor Speed Dial

Regulate the rotational speed of a DC motor proportionally using a potentiometer dial.

## Goal
Learn how to read an analog input voltage from a potentiometer dial, translate it to a PWM duty cycle (0-255) using `map()`, and drive motor speed.

## What You Will Build
As you turn the potentiometer dial on the canvas, the speed of the DC motor connected to D9 adjusts instantly. Turning the dial fully clockwise spins the motor at maximum speed, while turning it fully counter-clockwise stops it.

**Why A0 and D9?** Pin A0 measures the analog dial voltage. Pin D9 is a PWM output pin capable of varying average power to drive the motor speed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| DC Motor | `dc_motor` | Yes | Yes |
| NPN Transistor (e.g. PN2222) | `transistor` | Optional | Yes (critical driver) |
| Flyback Diode (e.g. 1N4007) | `diode` | Optional | Yes (critical protector) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Potentiometer | 1 | 5V | Positive voltage reference |
| Potentiometer | 2 (Wiper) | A0 | Analog signal connection |
| Potentiometer | 3 | GND | Ground reference |
| DC Motor | + | D9 | Control pin (PWM) |
| DC Motor | - | GND | Ground reference |

## Code
```cpp
const int POT_PIN   = A0;
const int MOTOR_PIN = 9; // Must be a PWM pin (3, 5, 6, 9, 10, 11)

void setup() {
  pinMode(MOTOR_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Potentiometer Motor Speed Controller Ready");
}

void loop() {
  int rawVal = analogRead(POT_PIN);
  
  // Map 0-1023 input range directly to 0-255 PWM duty cycle
  int motorSpeed = map(rawVal, 0, 1023, 0, 255);
  
  // Enforce speed safety limits
  motorSpeed = constrain(motorSpeed, 0, 255);
  
  analogWrite(MOTOR_PIN, motorSpeed);
  
  Serial.print("Pot Raw: ");
  Serial.print(rawVal);
  Serial.print(" | Motor Speed: ");
  Serial.println(motorSpeed);
  
  delay(100); // 10 Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Potentiometer**, and **DC Motor** onto the canvas.
2. Connect Potentiometer: **pin 1** to Arduino **5V**, **pin 2 (wiper)** to Arduino **A0**, and **pin 3** to Arduino **GND**.
3. Connect DC Motor: **+** to Arduino **D9** and **-** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the Potentiometer, adjust its knob value, and watch the DC Motor speed change.

## Expected Output

Terminal:
```
Potentiometer Motor Speed Controller Ready
Pot Raw: 0 | Motor Speed: 0
Pot Raw: 512 | Motor Speed: 127
Pot Raw: 1023 | Motor Speed: 255
...
```

### Expected Canvas Behavior

| Potentiometer Slider | Pin A0 Input | PWM Duty Cycle (D9) | Motor Speed |
| --- | --- | --- | --- |
| 0% (Left) | 0 | 0 (0V) | Stopped |
| 50% (Center) | 512 | 127 (2.5V average) | Medium Speed |
| 100% (Right) | 1023 | 255 (5V) | Full Speed |

The motor speed updates instantly as you adjust the potentiometer dial.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `int motorSpeed = map(rawVal, 0, 1023, 0, 255)` | Scales the 10-bit analog voltage range (0-1023) directly to the 8-bit output range (0-255) required for PWM speed control. |
| `analogWrite(MOTOR_PIN, motorSpeed)` | Adjusts the pulse width duty cycle on pin D9, regulating power. |

## Hardware & Safety Concept: Transistor Speed Regulation
PWM speed control works by switching the motor ON and OFF thousands of times per second (typically **490 Hz** or **980 Hz**).
- The ratio of ON time to OFF time is called the **duty cycle**.
- A 50% duty cycle means the motor gets full voltage half the time, which averages out to half-speed power.
Using a transistor as a high-speed switch allows the Arduino to regulate this average power safely without drawing excessive current.

## Try This! (Challenges)
1. **Minimum Speed Guard**: Low duty cycles (e.g. < 60) may not supply enough torque to overcome motor internal friction. Modify the code so that if the speed is between 1 and 60, it forces the motor speed to `60` (ensuring startup).
2. **Reverse Direction Switch**: Wire a button to D2. If pressed, invert the mapping so that turning the dial clockwise slows down the motor.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor stays fully off or on | Non-PWM pin used | Verify that the motor positive pin connects to D9, which supports hardware PWM. |
| Motor makes a high-pitched whine | Low frequency PWM friction | This is common at low duty cycles on physical motors. Try setting a minimum speed threshold in your code. |

## Mode Notes
These patterns (analog reads, `map()` scaling, and PWM `analogWrite()` drives) are supported by MbedO interpreted mode.

## Related Projects
- [14 - Dial LED Brightness](../beginner/14-dial-led-brightness.md)
- [46 - Potentiometer Servo](46-potentiometer-servo.md)
- [82 - Motor Start Stop](82-motor-start-stop.md)
