# 131 - Sound Activated Mobile Robot

Control a mobile robot's movement state using acoustic triggers, toggling between driving forward and stopping upon detecting a loud sound (clap) via a digital sound sensor.

## Goal
Learn how to interface digital sound threshold sensors, build a debounce-filtered state toggle mechanism without using blocking loops, and control DC motors based on acoustic inputs.

## What You Will Build
An acoustically controlled mobile robot. A digital sound sensor (microphone module with comparator) is connected to digital input pin `GPIO 9`. The ARIES board monitors this pin. When you clap, the sound sensor output temporarily pulses HIGH. The board detects this trigger, toggles the robot's state between "Stopped" and "Moving Forward", updates the motor driver outputs, and prints the current status to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Digital Sound Sensor Module | `sound_sensor` | Yes | Yes |
| L298N Motor Driver Module | `l298n` | Yes | Yes |
| 2x DC Toy Motors | `dc_motor` | Yes | Yes |
| External 6V-12V Power Source | `power_supply` | Optional | Yes (battery pack) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Sound Sensor | VCC | 3V3 | Red | Sensor operating voltage (3.3V) |
| Sound Sensor | GND | GND | Black | Ground reference |
| Sound Sensor | OUT (Digital Out) | GPIO 9 | Yellow | Sound trigger output |
| L298N Module | VCC / GND | 5V / GND | Red / Black | Motor driver power |
| L298N Module | IN1 / IN2 | GPIO 14 / 15 | Orange / Brown | Left motor control |
| L298N Module | IN3 / IN4 | GPIO 13 / 12 | Green / Blue | Right motor control |

> **Wiring tip:** Make sure you connect the digital output pin (OUT or DO) of the sound sensor to GPIO 9. Do not connect it to the analog output pin (AO) if your sensor board contains both.

## Code
```cpp
// Sound Activated Mobile Robot - VEGA ARIES v3
const int SOUND_PIN = 9;   // Digital Out of Sound Sensor on GPIO 9

const int IN1 = 14;        // Left Motor Forward
const int IN2 = 15;        // Left Motor Backward
const int IN3 = 13;        // Right Motor Forward
const int IN4 = 12;        // Right Motor Backward

bool robotMoving = false;  // State variable: true = moving forward, false = stopped

void setup() {
  // Configure sound sensor pin as input
  pinMode(SOUND_PIN, INPUT);

  // Configure motor driver pins as outputs
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  // Stop motors initially
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);

  // Initialize Serial Monitor
  Serial.begin(9600);
  Serial.println("Sound Activated Robot online. Waiting for clap...");
}

void loop() {
  // Read the sound sensor digital output
  int soundSignal = digitalRead(SOUND_PIN);

  // Check if a sound trigger (clap) is detected
  if (soundSignal == HIGH) {
    // Toggle the robot state
    robotMoving = !robotMoving;

    if (robotMoving) {
      Serial.println("Clap Detected! State: MOVING FORWARD");
      digitalWrite(IN1, HIGH);
      digitalWrite(IN2, LOW);
      digitalWrite(IN3, HIGH);
      digitalWrite(IN4, LOW);
    } 
    else {
      Serial.println("Clap Detected! State: STOPPED");
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, LOW);
      digitalWrite(IN3, LOW);
      digitalWrite(IN4, LOW);
    }

    // Debounce delay: ignore sound input for 500 ms
    // This prevents the same clap or echo from double-triggering the state
    delay(500);
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Sound Sensor**, **L298N Motor Driver**, and two **DC Toy Motors** onto the canvas.
2. Wire the L298N module power and outputs to the DC motors.
3. Wire L298N inputs to ARIES: **IN1** to **GPIO 14**, **IN2** to **GPIO 15**, **IN3** to **GPIO 13**, and **IN4** to **GPIO 12**.
4. Connect the Sound Sensor: **VCC** to **3V3**, **GND** to **GND**, and **OUT** to **GPIO 9**.
5. Paste the C++ code into the editor.
6. Select **Interpreted Mode** and click **Run**.
7. Click the simulated sound sensor output button on the canvas to simulate a clap.
8. Watch the motors change state from stopped to running forward, then back to stopped on the next click.

## Expected Output
Serial Monitor:
```
Sound Activated Robot online. Waiting for clap...
Clap Detected! State: MOVING FORWARD
Clap Detected! State: STOPPED
Clap Detected! State: MOVING FORWARD
```

## Expected Canvas Behavior
* When the simulation starts, the motors remain stopped.
* Clicking the simulated Sound Sensor trigger button once activates both virtual DC motors in the forward direction.
* Clicking the button a second time halts both motors.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(SOUND_PIN, INPUT)` | Registers GPIO 9 as a digital input pin to read triggers from the sound sensor's comparator. |
| `digitalRead(SOUND_PIN)` | Samples the sound sensor. Returns HIGH if the acoustic volume exceeds the sensor's threshold. |
| `robotMoving = !robotMoving` | Toggles the state variable using boolean NOT negation. |
| `delay(500)` | Suspends execution for half a second to filter out sound reflections and echoes. |

## Hardware & Safety Concept
* **Microphone Comparator Thresholds**: Sound sensors use an electret microphone capsule and an onboard operational amplifier comparator (like the LM393). The board features a small multi-turn trim potentiometer. Turning this potentiometer adjusts the voltage reference threshold. Adjust it so the sensor's indicator LED stays off in quiet environments, but blinks briefly when you clap your hands.
* **Debouncing Sound Inputs**: Unlike clean mechanical switch presses, claps create a messy series of high-frequency pressure peaks that can last for 50–200 ms. Without a software filter or a dead-time window (the 500 ms delay), the micro-controller will register the single clap as 10 to 20 separate triggers, causing the robot to rapidly toggle states and appear to react erratically.

## Try This! (Challenges)
1. **Double Clap Turn**: Modify the code to count the number of claps within a 1-second window. A single clap toggles moving/stopping, while a double clap turns the robot 90 degrees to the right. (Do not use `for` or `while` loops!)
2. **Acoustic Warning Buzzer**: Connect an active buzzer to GPIO 14. Program it to sound a short chirp (100 ms) whenever a clap is successfully registered, providing instant feedback.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The robot toggles states constantly, even in a quiet room | Threshold pot on the sound module is set too sensitive | Turn the physical potentiometer screw counter-clockwise to reduce sensitivity. |
| The robot ignores claps completely | Threshold is set too high or incorrect output wire used | Turn the physical potentiometer clockwise to increase sensitivity. Verify that the ARIES pin is connected to the DO (Digital Out) of the sound sensor. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [30 - Sound Sensor Clapper](../beginner/30-sound-sensor-clapper.md)
- [49 - Sound Sensor Threshold Alarm](../beginner/49-sound-sensor-threshold-alarm.md)
