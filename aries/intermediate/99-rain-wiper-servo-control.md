# 99 - Rain Wiper Servo Control

Create an automatic rain-sensing windshield wiper system using an analog rain sensor and a servo motor on the VEGA ARIES v3 board.

## Goal
Learn how to read analog inputs from a resistive rain sensor, process environmental moisture levels, and program a loop-free servo sweeping state machine that activates when rain is detected and parks the wiper when dry.

## What You Will Build
An automated wiper system. When the rain sensor is dry, the servo motor remains stationary in its parked position (0°). When water droplets touch the sensor plate, the servo automatically begins sweeping back and forth to simulate a vehicle's windshield wiper.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Analog Rain Sensor | `rain_sensor` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Rain Sensor | VCC | 3V3 | Red | Sensor logic power (3.3V) |
| Rain Sensor | GND | GND | Black | Ground reference |
| Rain Sensor | AO | ADC1 (GP27) | White | Analog moisture level output |
| Servo Motor | Signal | GPIO 13 | Yellow | PWM control signal |
| Servo Motor | VCC | 5V | Red | Servo power supply |
| Servo Motor | GND | GND | Black | Ground reference |

> **Wiring tip:** The rain sensor plate connects to its comparator module, which outputs an analog signal. Keep the servo power line (5V) separate from the sensor power line (3.3V) to prevent signal noise on the analog read.

## Code
```cpp
#include <Servo.h>

const int RAIN_PIN = 27;     // ADC1 is GP27
const int SERVO_PIN = 13;    // Servo Signal on GPIO 13

Servo wiperServo;
int servoAngle = 0;
int sweepDirection = 5;      // 5-degree steps
const int WET_THRESHOLD = 2500; // Lower value = higher moisture

void setup() {
  wiperServo.attach(SERVO_PIN);
  wiperServo.write(0); // Park the wiper at 0 degrees
}

void loop() {
  int rainValue = analogRead(RAIN_PIN);

  // Resistive rain sensors output LOWER voltage values as they get wetter
  if (rainValue < WET_THRESHOLD) {
    // Sweep state update (run iteratively without loops)
    servoAngle += sweepDirection;
    
    // Reverse movement at boundaries
    if (servoAngle >= 180 || servoAngle <= 0) {
      sweepDirection = -sweepDirection;
    }
    
    wiperServo.write(servoAngle);
  } else {
    // Park the wiper at 0 degrees when dry
    if (servoAngle != 0) {
      servoAngle = 0;
      wiperServo.write(servoAngle);
    }
  }

  delay(35); // Control the sweep speed
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Analog Rain Sensor**, and **Servo Motor** components onto the canvas.
2. Wire the Rain Sensor: **VCC** to **3V3**, **GND** to **GND**, and **AO** to **ADC1 (GP27)**.
3. Wire the Servo Motor: **Signal** to **GPIO 13**, **VCC** to **5V**, and **GND** to **GND**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Rain Sensor Raw ADC: 4095 (Dry - Wiper Parked)
...
Rain Sensor Raw ADC: 1200 (Wet - Sweeping active)
```

## Expected Canvas Behavior
* Decreasing the analog slider on the simulated rain sensor represents rain droplets contacting the sensor plate.
* When the value falls below 2500, the servo motor starts sweeping back and forth smoothly.
* Resetting the slider to the top (dry) stops the sweep and rotates the servo back to 0°.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `wiperServo.attach(13)` | Sets up the PWM channel on pin GPIO 13 for servo control. |
| `analogRead(RAIN_PIN)` | Samples the resistive voltage level from the rain module. |
| `rainValue < WET_THRESHOLD` | Evaluates if water presence has dropped the resistance below the threshold. |
| `servoAngle += sweepDirection` | Iteratively shifts the servo position by a small step to avoid blocks or delays. |
| `wiperServo.write(0)` | Forces the motor back to the parking position when sensor becomes dry. |

## Hardware & Safety Concept
* **Resistive Electrolysis**: Applying constant DC voltage to a resistive sensor plate submerged in water causes rapid oxidation/electrolysis of the copper traces. In professional applications, the sensor VCC is connected to a digital output pin and only powered ON immediately before reading, then powered OFF to prolong lifespan.
* **Servo Peak Current**: A sweeping wiper mechanical link can bind and stall the motor. Ensure the power source is capable of sustaining stall currents to prevent ARIES reset cycles.

## Try This! (Challenges)
1. **Rain Intensity Speed Control**: Adjust the sweep speed dynamically based on the rain level. For heavy rain (low ADC value), make the wiper sweep faster by reducing the delay to 15 ms.
2. **Double Sweep Cycle**: Modify the logic so that when rain is detected, the wiper is guaranteed to complete at least one full 0°-180°-0° sweep cycle before checking the dryness state again.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Wiper does not move when wet | Threshold set too low | Check the raw analog readings. Some sensors output different base values. Adjust the threshold accordingly. |
| Servo vibrates or moves in steps | Inadequate power supply | Ensure the servo VCC is connected to a dedicated 5V rail and that grounds are shared. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - Servo Motor 180 Degree Sweep](51-servo-motor-180-degree-sweep.md)
- [97 - Joystick Dual Axis Servo (Pan & Tilt)](97-joystick-dual-axis-servo.md)
