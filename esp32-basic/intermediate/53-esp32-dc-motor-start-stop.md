# 53 - ESP32 DC Motor Start/Stop

Control a DC motor through an L298N dual H-bridge motor driver — starting and stopping the motor with direction control using two GPIO pins.

## Goal
Learn how an L298N H-bridge driver controls a DC motor's direction and enable state, and how to wire it safely so the ESP32's 3.3 V logic drives a 5–12 V motor circuit.

## What You Will Build
An L298N motor driver with a DC motor on Output A. GPIO 18 and GPIO 19 control direction (IN1/IN2); GPIO 5 enables the channel (ENA). The motor runs forward for 3 seconds, stops for 1 second, runs reverse for 3 seconds, then repeats.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motor (3–12 V) | `dc_motor` | Yes | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 | GPIO18 | Yellow | Direction bit 1 |
| L298N Module | IN2 | GPIO19 | Green | Direction bit 2 |
| L298N Module | ENA | GPIO5 | Orange | Channel A enable — HIGH = enabled |
| L298N Module | GND | GND | Black | Shared ground with ESP32 |
| L298N Module | 12V (VCC) | External 6–12 V supply + | Red | Motor power supply |
| L298N Module | GND (power) | External supply − | Black | Motor supply ground |
| DC Motor | Terminal A | OUT1 | Blue | Motor lead 1 |
| DC Motor | Terminal B | OUT2 | White | Motor lead 2 |

> **Wiring tip:** The L298N has its own onboard 5 V regulator — leave the 5V jumper on the module and it will supply 5 V to its logic from the motor supply. The ESP32 and L298N must share a common GND — connect a wire from ESP32 GND to the L298N GND terminal. Never connect the motor supply directly to ESP32 VCC; keep motor and logic power separate.

## Code
```cpp
// DC Motor Start/Stop via L298N H-bridge
const int IN1 = 18;   // Direction bit 1
const int IN2 = 19;   // Direction bit 2
const int ENA = 5;    // Channel A enable

void motorForward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(ENA, HIGH);
  Serial.println("Motor: FORWARD");
}

void motorReverse() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(ENA, HIGH);
  Serial.println("Motor: REVERSE");
}

void motorStop() {
  digitalWrite(ENA, LOW);   // Disable channel — motor coasts to stop
  Serial.println("Motor: STOP");
}

void setup() {
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(ENA, OUTPUT);
  motorStop();   // Safe initial state
  Serial.begin(115200);
  Serial.println("DC Motor Start/Stop ready.");
}

void loop() {
  motorForward();
  delay(3000);

  motorStop();
  delay(1000);

  motorReverse();
  delay(3000);

  motorStop();
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N Motor Driver**, and **DC Motor** onto the canvas.
2. Connect IN1 to **GPIO18**, IN2 to **GPIO19**, ENA to **GPIO5**.
3. Connect Motor to L298N **OUT1/OUT2**.
4. Paste the code and click **Run**.
5. Observe the motor widget spinning forward, stopping, reversing, and stopping.

## Expected Output
Serial Monitor:
```
DC Motor Start/Stop ready.
Motor: FORWARD
Motor: STOP
Motor: REVERSE
Motor: STOP
```

## Expected Canvas Behavior
* Motor widget spins clockwise for 3 seconds.
* Motor stops (no spin) for 1 second.
* Motor spins counter-clockwise for 3 seconds.
* Cycle repeats.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `IN1=HIGH, IN2=LOW` | Sets the H-bridge to route current through the motor in the forward direction. |
| `IN1=LOW, IN2=HIGH` | Reverses the H-bridge logic — current flows through the motor in the opposite direction. |
| `digitalWrite(ENA, LOW)` | Disables channel A — motor terminals float and motor coasts to a stop. |
| `motorStop()` in `setup()` | Ensures the motor starts in a safe stopped state on power-up. |

## Hardware & Safety Concept: H-Bridge Motor Drivers
An H-bridge is four switches arranged in an H shape around a motor. By closing different pairs of switches, current flows through the motor in either direction. The L298N integrates two independent H-bridges (channels A and B) with logic-level input pins. **Critical safety rule:** Never set IN1=HIGH and IN2=HIGH simultaneously — this creates a shoot-through condition, connecting the supply directly to GND through both switches and destroying the driver chip. The code avoids this by always setting ENA LOW before changing IN1/IN2 direction. Flyback diodes inside the L298N protect against inductive voltage spikes when the motor coil is de-energised.

## Try This! (Challenges)
1. **Button control**: Add a button on GPIO 4 — press to cycle FORWARD → STOP → REVERSE → STOP.
2. **Speed display**: Use `ledcWrite` on ENA for full/half speed states and print the selected speed.
3. **Brake mode**: Instead of `ENA=LOW` (coast), try `IN1=HIGH, IN2=HIGH, ENA=HIGH` — this applies an electrical braking force.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor does not spin | ENA not driven HIGH | Verify GPIO 5 is wired to ENA and `digitalWrite(ENA, HIGH)` is called before running |
| Motor runs in one direction only | IN1 and IN2 wired to same GPIO | Check IN1 → GPIO18, IN2 → GPIO19 independently |
| L298N gets very hot | Motor stalling or supply too high | Reduce motor supply voltage; ensure motor is free to spin |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [54 - ESP32 DC Motor Speed Scaling](54-esp32-dc-motor-speed-scaling.md)
- [55 - ESP32 DC Motor Direction Toggle](55-esp32-dc-motor-direction-toggle.md)
- [11 - ESP32 Relay Module Control](../beginner/11-esp32-relay-module-control.md)
