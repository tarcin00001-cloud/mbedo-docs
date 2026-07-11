# 188 - ESP32 MCPWM Motor Control

Build an advanced motor driver controller on the ESP32 that configures the internal Motor Control PWM (MCPWM) hardware peripheral to drive an H-Bridge L298N DC motor, using a potentiometer to adjust speed and direction.

## Goal
Learn how to configure the ESP32's dedicated MCPWM hardware peripheral, map outputs to MCPWM generator channels, and configure duty cycles.

## What You Will Build
An L298N driver controls a DC motor (IN1: 18, IN2: 19). A potentiometer is on GPIO 34. The ESP32 does not use `digitalWrite` or `ledc` for motor control. Instead, it configures MCPWM Unit 0 with generator channels 0A (GPIO 18) and 0B (GPIO 19) to control motor speed and direction.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motor | `dc_motor` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| External Power Supply (6-12V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 (Generator 0A) | GPIO18 | Yellow | Forward MCPWM output |
| L298N Module | IN2 (Generator 0B) | GPIO19 | Green | Reverse MCPWM output |
| L298N Module | ENA | 5V (or jumpered)| Red | Enable jumper tied HIGH |
| Potentiometer | Wiper | GPIO34 | Yellow | Speed control wiper |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** When using MCPWM, connect IN1 directly to GPIO 18 and IN2 to GPIO 19. Ensure the L298N ENA jumper is installed (tying it HIGH to 5V) to enable the H-bridge.

## Code
```cpp
// MCPWM Motor Control (Hardware MCPWM Driver)
#include <Arduino.h>
#include "driver/mcpwm.h"

const int POT_PIN = 34;

// MCPWM GPIO Pin Assignments
#define GPIO_PWM0A_OUT 18 // Pin connected to IN1
#define GPIO_PWM0B_OUT 19 // Pin connected to IN2

void initMCPWM() {
  Serial.println("Configuring MCPWM peripheral...");
  
  // 1. Map GPIO pins to MCPWM generator signals
  mcpwm_gpio_init(MCPWM_UNIT_0, MCPWM0A, GPIO_PWM0A_OUT);
  mcpwm_gpio_init(MCPWM_UNIT_0, MCPWM0B, GPIO_PWM0B_OUT);
  
  // 2. Configure MCPWM parameters
  mcpwm_config_t pwm_config;
  pwm_config.frequency = 1000;             // Frequency = 1 kHz
  pwm_config.cmpr_a = 0.0;                 // Initial duty cycle for Generator A (0%)
  pwm_config.cmpr_b = 0.0;                 // Initial duty cycle for Generator B (0%)
  pwm_config.counter_mode = MCPWM_UP_COUNTER;
  pwm_config.duty_mode = MCPWM_DUTY_MODE_0; // Active HIGH duty cycle
  
  // 3. Initialize MCPWM unit 0 timer 0
  mcpwm_init(MCPWM_UNIT_0, MCPWM_TIMER_0, &pwm_config);
  
  Serial.println("MCPWM initialization complete.");
}

void setup() {
  Serial.begin(115200);
  pinMode(POT_PIN, INPUT);
  
  initMCPWM();
}

void loop() {
  // Read potentiometer (0 to 4095)
  // Center (2048) = stopped
  int potVal = analogRead(POT_PIN);
  
  float dutyCycle = 0.0;
  
  // 4. Update Speed and Direction
  if (potVal > 2100) {
    // Forward: Generator A active, Generator B off (0% duty)
    // Scale duty cycle: 2100-4095 -> 0% to 100%
    dutyCycle = (float)(potVal - 2100) * 100.0 / 1995.0;
    dutyCycle = constrain(dutyCycle, 0.0, 100.0);
    
    // Command MCPWM duty cycles
    mcpwm_set_duty(MCPWM_UNIT_0, MCPWM_TIMER_0, MCPWM_OPR_A, dutyCycle);
    mcpwm_set_duty(MCPWM_UNIT_0, MCPWM_TIMER_0, MCPWM_OPR_B, 0.0);
    
    Serial.print("Direction: FORWARD | Duty: "); Serial.print(dutyCycle, 1); Serial.println(" %");
  } 
  else if (potVal < 2000) {
    // Reverse: Generator B active, Generator A off (0% duty)
    // Scale duty cycle: 2000-0 -> 0% to 100%
    dutyCycle = (float)(2000 - potVal) * 100.0 / 2000.0;
    dutyCycle = constrain(dutyCycle, 0.0, 100.0);
    
    mcpwm_set_duty(MCPWM_UNIT_0, MCPWM_TIMER_0, MCPWM_OPR_A, 0.0);
    mcpwm_set_duty(MCPWM_UNIT_0, MCPWM_TIMER_0, MCPWM_OPR_B, dutyCycle);
    
    Serial.print("Direction: REVERSE | Duty: "); Serial.print(dutyCycle, 1); Serial.println(" %");
  } 
  else {
    // Stop: Both generators off
    mcpwm_set_duty(MCPWM_UNIT_0, MCPWM_TIMER_0, MCPWM_OPR_A, 0.0);
    mcpwm_set_duty(MCPWM_UNIT_0, MCPWM_TIMER_0, MCPWM_OPR_B, 0.0);
    
    Serial.println("Direction: STOPPED | Duty: 0.0 %");
  }
  
  delay(150); // 7Hz log rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, **DC Motor**, and **Potentiometer** onto the canvas.
2. Wire IN1/IN2 to **GPIO18/GPIO19**, and Potentiometer to **GPIO34**.
3. Paste the code and click **Run**.
4. Slide the potentiometer widget right. Watch the motor rotate clockwise.
5. Slide the potentiometer widget left. Watch the motor rotate counter-clockwise.

## Expected Output
Serial Monitor:
```
Configuring MCPWM peripheral...
MCPWM initialization complete.
Direction: FORWARD | Duty: 45.2 %
Direction: REVERSE | Duty: 62.1 %
Direction: STOPPED | Duty: 0.0 %
```

## Expected Canvas Behavior
* At boot, the motor is stopped.
* Sliding the potentiometer slider past 2100 spins the motor clockwise.
* Sliding it below 2000 spins the motor counter-clockwise.
* The motor speed changes dynamically in response to the slider.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `mcpwm_gpio_init(...)` | Maps the GPIO pins directly to the MCPWM hardware generators. |
| `pwm_config.frequency` | Sets the PWM carrier frequency to 1 kHz. |
| `mcpwm_set_duty(...)` | Adjusts the duty cycle of the specified generator (OPR_A or OPR_B) directly. |

## Hardware & Safety Concept: Motor Control PWM (MCPWM) vs LEDC
Standard PWM drivers (like LEDC) are designed for simple applications like dimming LEDs. For driving H-bridges and motor coils, they lack critical safety features. The ESP32 contains **two MCPWM units**. These units support generating complementary PWM signals with deadtime insertion to prevent short circuits, and can automatically shut down the outputs on external fault inputs (like limit switches) to protect hardware.

## Try This! (Challenges)
1. **Emergency Limit Stop switch**: Add a button on GPIO 4 that triggers an MCPWM fault input to shut off the motor instantly.
2. **Complementary drive**: Configure the MCPWM to run in complementary mode, driving both generators with inverted signals.
3. **Dynamic Frequency scaling**: Adjust the PWM carrier frequency dynamically from 500 Hz to 20 kHz to test motor hum noise.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor does not spin | ENA pin disabled | Ensure the ENA jumper is installed on the L298N board |
| Motor spins in one direction only | Pin mapping conflict | Verify that IN1 is connected to GPIO 18 and IN2 to GPIO 19 |
| Motor hums but does not spin at low duty | PWM frequency too high | Reduce the frequency in `pwm_config.frequency` to 500 Hz or 1 kHz |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [123 - ESP32 Robot Speed Control](../intermediate/123-esp32-robot-speed-control.md)
- [151 - ESP32 DC Motor PID Speed Controller](151-esp32-dc-motor-pid-speed-controller.md)
- [187 - ESP32 Hardware Pulse Counter](187-esp32-hardware-pulse-counter.md)
