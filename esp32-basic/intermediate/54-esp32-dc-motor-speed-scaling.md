# 54 - ESP32 DC Motor Speed Scaling

Use the ESP32's LEDC hardware PWM to control the speed of a DC motor via the L298N ENA pin — from fully stopped through multiple speed steps to full speed.

## Goal
Learn how to use `ledcWrite()` on the motor driver's enable pin to produce variable speed by varying the PWM duty cycle, and understand how speed maps to duty cycle.

## What You Will Build
An L298N motor driver with a DC motor on Channel A. GPIO 18/19 set forward direction; GPIO 5 (ENA) receives PWM from LEDC channel 0. The code ramps speed from 0% to 100% in steps, holds, then ramps back down.

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
| L298N Module | IN1 | GPIO18 | Yellow | Direction bit 1 — fixed HIGH for forward |
| L298N Module | IN2 | GPIO19 | Green | Direction bit 2 — fixed LOW for forward |
| L298N Module | ENA | GPIO5 | Orange | PWM speed control input |
| L298N Module | GND | GND | Black | Shared ground with ESP32 |
| L298N Module | 12V (VCC) | External 6–12 V supply + | Red | Motor power supply |
| L298N Module | GND (power) | External supply − | Black | Motor supply ground |
| DC Motor | Terminal A | OUT1 | Blue | Motor lead 1 |
| DC Motor | Terminal B | OUT2 | White | Motor lead 2 |

> **Wiring tip:** Remove the ENA jumper from the L298N module (the short jumper that hard-wires ENA HIGH) and connect GPIO 5 to the ENA pin directly. Without removing the jumper, ENA is always HIGH and speed control has no effect.

## Code
```cpp
// DC Motor Speed Scaling via LEDC PWM on ENA
const int IN1      = 18;
const int IN2      = 19;
const int ENA_PIN  = 5;
const int PWM_CHAN = 0;
const int PWM_FREQ = 1000;   // 1 kHz — audible motor tone is above ~200 Hz
const int PWM_RES  = 8;      // 8-bit: 0–255

void setSpeed(int duty) {
  duty = constrain(duty, 0, 255);
  ledcWrite(PWM_CHAN, duty);
  int pct = duty * 100 / 255;
  Serial.print("Speed: "); Serial.print(pct); Serial.println("%");
}

void setup() {
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);

  // Configure LEDC for ENA
  ledcSetup(PWM_CHAN, PWM_FREQ, PWM_RES);
  ledcAttachPin(ENA_PIN, PWM_CHAN);

  // Set forward direction
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);

  Serial.begin(115200);
  Serial.println("DC Motor Speed Control ready.");
}

void loop() {
  // Ramp UP: 0% → 100% in 25% steps
  Serial.println("--- Ramping UP ---");
  for (int duty = 0; duty <= 255; duty += 64) {
    setSpeed(duty);
    delay(1500);
  }
  setSpeed(255);   // Ensure full speed
  delay(2000);

  // Ramp DOWN: 100% → 0% in 25% steps
  Serial.println("--- Ramping DOWN ---");
  for (int duty = 255; duty >= 0; duty -= 64) {
    setSpeed(duty);
    delay(1500);
  }
  setSpeed(0);     // Ensure full stop
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, and **DC Motor** onto the canvas.
2. Connect IN1 → **GPIO18**, IN2 → **GPIO19**, ENA → **GPIO5**.
3. Paste the code and click **Run**.
4. Observe the motor widget slowly accelerating and decelerating in the canvas.

## Expected Output
Serial Monitor:
```
DC Motor Speed Control ready.
--- Ramping UP ---
Speed: 0%
Speed: 25%
Speed: 50%
Speed: 75%
Speed: 100%
--- Ramping DOWN ---
Speed: 100%
Speed: 75%
Speed: 50%
Speed: 25%
Speed: 0%
```

## Expected Canvas Behavior
* Motor widget starts stationary, then spins progressively faster.
* At 100% the motor is at maximum speed.
* It slows progressively back to stopped, then the cycle repeats.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `ledcSetup(PWM_CHAN, PWM_FREQ, PWM_RES)` | Configures LEDC channel 0 at 1 kHz, 8-bit resolution for the ENA pin. |
| `ledcAttachPin(ENA_PIN, PWM_CHAN)` | Binds GPIO 5 to the LEDC channel. |
| `ledcWrite(PWM_CHAN, duty)` | Sets the PWM duty on ENA — 0 = stop, 255 = full speed. |
| `duty += 64` | Increments by 64 per step ≈ 25% per step (64/255 ≈ 25%). |
| `constrain(duty, 0, 255)` | Prevents invalid duty values outside the 8-bit range. |

## Hardware & Safety Concept: PWM Speed Control of DC Motors
A DC motor's speed is proportional to the **average voltage** across it. PWM achieves variable average voltage by switching the supply on and off rapidly. At 50% duty cycle and 12 V supply, the average voltage is 6 V — the motor runs at roughly half speed. The L298N ENA pin is internally connected to the enable input of the H-bridge's driver transistors. When PWM is applied to ENA, the H-bridge switches on and off at the PWM frequency, varying the average motor voltage. The motor's armature inductance smooths the current ripple, so the motor experiences a near-constant current at the average level — this is called **chopper control** and is the basis of all modern motor drives.

## Try This! (Challenges)
1. **Potentiometer speed**: Read GPIO 34 ADC and map it directly to the motor duty cycle for real-time manual speed control.
2. **Smooth ramp**: Change the step from 64 to 1 with a 10 ms delay — creating a slow, smooth acceleration curve.
3. **Speed + direction**: Combine Project 53 (direction toggle) with this project — forward ramp up, reverse ramp down.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor always runs at full speed | ENA jumper still installed on L298N | Remove the ENA header jumper and wire GPIO 5 directly |
| Motor does not spin at any duty | PWM channel not attached or wrong pin | Verify `ledcAttachPin(ENA_PIN, PWM_CHAN)` is called in setup |
| Squealing noise from motor | PWM frequency too low | Increase `PWM_FREQ` to 5000 or higher |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [53 - ESP32 DC Motor Start/Stop](53-esp32-dc-motor-start-stop.md)
- [55 - ESP32 DC Motor Direction Toggle](55-esp32-dc-motor-direction-toggle.md)
- [67 - ESP32 Potentiometer Speed DC Motor](67-esp32-potentiometer-speed-dc-motor.md)
