# 101 - Automatic Barrier Gate

Create a smart automated parking barrier or security gate system using an infrared (IR) proximity sensor, a servo motor, and an I2C LCD display.

## Goal
Learn how to interface digital infrared sensors, operate a servo motor dynamically based on object detection, and update status messages on an LCD screen.

## What You Will Build
An automatic vehicle entry barrier. When an approaching vehicle is detected by the IR sensor, the system updates the LCD to show `GATE: OPEN` and rotates the servo motor to 90 degrees to raise the barrier. Once the vehicle clears the sensor, the barrier closes (servo returns to 0 degrees) and updates the display.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| IR Proximity / Obstacle Sensor | `ir_obstacle` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Sensor | VCC | 3V3 | Red | Sensor operating voltage (3.3V) |
| IR Sensor | GND | GND | Black | Ground reference |
| IR Sensor | OUT | GPIO 17 | Green | Digital obstacle signal (Active LOW) |
| Servo Motor | Signal | GPIO 13 | Yellow | PWM control signal |
| Servo Motor | VCC | 5V | Red | Servo power supply (5V) |
| Servo Motor | GND | GND | Black | Ground reference |
| I2C LCD | VCC | 5V | Red | LCD power supply (5V) |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** Standard digital IR proximity sensors output a `LOW` logic level when an obstacle is detected within range, and `HIGH` when the path is clear. Verify your sensor's logic type.

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Servo.h>

const int IR_PIN = 17;     // IR sensor on GPIO 17
const int SERVO_PIN = 13;  // Servo on GPIO 13

Servo gateServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

int lastState = -1;

void setup() {
  Wire.begin();
  
  pinMode(IR_PIN, INPUT);
  gateServo.attach(SERVO_PIN);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Barrier System");
  
  // Start with gate closed
  gateServo.write(0);
}

void loop() {
  // Read state: Active LOW (0 = Obstacle detected, 1 = Clear)
  int sensorState = digitalRead(IR_PIN);

  // Update only on state change to avoid screen flicker and redundant commands
  if (sensorState != lastState) {
    if (sensorState == LOW) { 
      gateServo.write(90); // Open the barrier
      lcd.setCursor(0, 1);
      lcd.print("GATE: OPEN      ");
    } else { 
      gateServo.write(0);  // Close the barrier
      lcd.setCursor(0, 1);
      lcd.print("GATE: CLOSED    ");
    }
    lastState = sensorState;
  }
  delay(100); // Polling delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **IR Proximity Sensor**, **Servo Motor**, and **I2C LCD Display** components onto the canvas.
2. Wire the IR Sensor: **VCC** to **3V3**, **GND** to **GND**, and **OUT** to **GPIO 17**.
3. Wire the Servo: **Signal** to **GPIO 13**, **VCC** to **5V**, and **GND** to **GND**.
4. Wire the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Obstacle Detected - Opening Gate.
Obstacle Cleared - Closing Gate.
```

## Expected Canvas Behavior
* Triggering the simulated IR sensor (simulating a car in front of the gate) causes the servo motor to rotate to 90 degrees.
* The LCD screen immediately changes row 2 to read `GATE: OPEN`.
* Untriggering the IR sensor closes the servo back to 0 degrees and changes the LCD text back to `GATE: CLOSED`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(IR_PIN, INPUT)` | Configures pin GPIO 17 as a digital input to read the IR sensor. |
| `gateServo.attach(SERVO_PIN)` | Configures GPIO 13 to drive the servo motor with PWM. |
| `digitalRead(IR_PIN)` | Samples the current binary logic state of the IR sensor. |
| `gateServo.write(90)` | Sets the target duty cycle on the servo PWM line to shift to 90 degrees. |

## Hardware & Safety Concept
* **Infrared Sensor Interference**: IR sensors work by sending a 38kHz infrared beam and measuring reflected light. External infrared light sources (such as direct sunlight, incandescent bulbs, or highly reflective surfaces) can saturate the receiver photodiode, causing false triggers.
* **Servo Mechanical Limits**: Driving a servo past its physical physical limits (typically 0° or 180°) causes the motor to strain against its internal endstops. This draws high currents and will strip the nylon gears or overheat the motor driver. Always keep programmed angles within safe mechanical boundaries.

## Try This! (Challenges)
1. **Closing Delay**: Instead of closing the gate immediately, wait for 3 seconds (`delay(3000)`) after the IR sensor is cleared before writing 0° to the servo, allowing time for a car to pass.
2. **Access Denied LED**: Connect a warning LED to GPIO 15. If the gate remains open for more than 5 seconds (indicating a blocked gate or tailgating), flash the warning LED.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The gate never opens when something is in front | Sensitivity threshold misadjusted | Adjust the onboard sensitivity potentiometer on the physical IR module until the indicator LED lights up only when objects are near. |
| Gate operates backward (opens when clear) | Sensor output is active high | Invert the logic check: `sensorState == HIGH` instead of `LOW` in the code. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - Servo Motor 180 Degree Sweep](51-servo-motor-180-degree-sweep.md)
- [58 - 16x2 I2C LCD Hello World](58-16x2-i2c-lcd-hello-world.md)
