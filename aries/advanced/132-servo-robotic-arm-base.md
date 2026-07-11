# 132 - Servo Robotic Arm Base Rotation

Control the angular position of a robotic arm's base rotation servo motor using a rotary potentiometer.

## Goal
Learn how to map analog input ranges to angular commands, write servo driver code using the Servo library, and monitor position feedback via the Serial Monitor.

## What You Will Build
A single-axis robotic joint controller representing a robotic arm base. A rotary potentiometer is connected to the analog input pin `ADC0` (GP26), and a hobby servo motor is connected to digital output pin `GPIO 13`. As you turn the potentiometer, the ARIES board reads the analog voltage, maps the 12-bit value to a range of 0° to 180°, updates the servo motor position, and prints the coordinates to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Hobby Servo Motor | `servo` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | VCC | 3V3 | Red | Analog reference voltage |
| Potentiometer | Output / Wiper | ADC0 (GP26) | Yellow | Joint input position |
| Potentiometer | GND | GND | Black | Ground reference |
| Hobby Servo | VCC | 5V | Red | Servo power supply (5V required) |
| Hobby Servo | GND | GND | Black | Ground reference |
| Hobby Servo | Signal | GPIO 13 | Yellow | Servo control line |

> **Wiring tip:** Connect the potentiometer to 3.3V, not 5V. The analog input pins on the ARIES board are rated for 3.3V maximum. Connecting the potentiometer to 5V will exceed the ADC input range and could damage the ARIES board.

## Code
```cpp
// Robotic Arm Base Rotation - VEGA ARIES v3
#include <Servo.h>

const int POT_PIN = GP26;   // Potentiometer on ADC0
const int SERVO_PIN = 13;   // Base Servo on GPIO 13

Servo baseServo;

void setup() {
  // Initialize primary USB Serial for position feedback
  Serial.begin(9600);
  
  // Register the servo motor pin
  baseServo.attach(SERVO_PIN);
  
  Serial.println("Robotic Arm Base Joint controller online.");
}

void loop() {
  // Read the 12-bit potentiometer value (0 - 4095)
  int potVal = analogRead(POT_PIN);

  // Map the 0-4095 ADC range to 0-180 degrees for the servo
  int targetAngle = map(potVal, 0, 4095, 0, 180);

  // Drive the servo to the mapped angle
  baseServo.write(targetAngle);

  // Print diagnostics to the Serial Monitor
  Serial.print("ADC Input: ");
  Serial.print(potVal);
  Serial.print(" | Command Angle: ");
  Serial.print(targetAngle);
  Serial.println(" deg");

  // Wait 20 ms to allow the servo to physically rotate and prevent buffer flooding
  delay(20);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, a **Potentiometer**, and a **Servo** onto the canvas.
2. Wire the Potentiometer: **VCC** to **3V3**, **GND** to **GND**, and the center pin **Output** to **GP26 (ADC0)**.
3. Wire the Servo: **VCC** to **5V**, **GND** to **GND**, and **Signal** to **GPIO 13**.
4. Paste the C++ code into the editor.
5. Select **Interpreted Mode** and click **Run**.
6. Rotate the virtual potentiometer slider. Watch the servo horn position rotate synchronously between 0 and 180 degrees.

## Expected Output
Serial Monitor:
```
Robotic Arm Base Joint controller online.
ADC Input: 2048 | Command Angle: 90 deg
ADC Input: 1024 | Command Angle: 45 deg
ADC Input: 4095 | Command Angle: 180 deg
```

## Expected Canvas Behavior
* As the potentiometer slider is rotated, the virtual servo widget's shaft rotates to match the corresponding mapped angle in real-time.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <Servo.h>` | Includes the standard library to interface servo motors. |
| `baseServo.attach(13)` | Informs the Servo library that the servo signal wire is plugged into GPIO 13. |
| `analogRead(GP26)` | Reads the analog voltage from the potentiometer wiper (0V to 3.3V returned as 0 to 4095). |
| `map(potVal, 0, 4095, 0, 180)` | Linearly maps the 12-bit ADC value into degrees of rotation (0 to 180). |
| `baseServo.write(targetAngle)` | Outputs the calculated PWM pulse to rotate the servo to the desired angle. |

## Hardware & Safety Concept
* **Analog Resolution (ADC)**: The VEGA ARIES v3 board features a 12-bit Analog-to-Digital Converter (ADC). This means it divides the analog input range of 0V to 3.3V into 4,096 discrete steps (0 to 4095). A reading of 0 corresponds to 0V, 2048 to 1.65V, and 4095 to 3.3V. This high resolution allows for very precise motor positioning.
* **Servo Inrush Currents**: High-torque servos (such as metal-gear MG996R units used in robotic arms) can draw inrush current spikes of over 1A when starting to rotate. If powered directly from the ARIES 5V pin, these spikes can drop the board's voltage rail, triggering the brownout reset circuit. For multi-joint arms, always power the servos from an external 5V power supply.

## Try This! (Challenges)
1. **Limited Joint Range**: Robotic arms often have mechanical obstructions. Modify the mapping range so that the servo is limited to rotating between 30° and 150°, preventing it from colliding with structural mounts.
2. **Speed Limiter (Smooth Drive)**: Instead of writing the target angle directly, increment or decrement the active position by 1 degree toward the target angle on each loop execution to smooth out sudden jerky movements.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not reach full 180 degrees or behaves erratically | Potentiometer connected to 5V instead of 3.3V | Ensure the potentiometer VCC connects to 3.3V to match the ARIES ADC reference level. |
| Servo only moves to extreme ends (0 or 180) and stays there | Potentiometer wiper wired to wrong pin or broken track | Double check that the middle pin of the potentiometer is wired to ARIES pin GP26 (ADC0). |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [45 - Potentiometer Servo Sweep](../beginner/45-potentiometer-servo-sweep.md)
- [133 - 2-Axis Robotic Arm Joint Controller](133-2-axis-robotic-arm-joint-controller.md)
