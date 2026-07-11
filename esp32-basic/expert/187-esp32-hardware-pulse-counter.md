# 187 - ESP32 Hardware Pulse Counter

Build a high-speed rotational encoder monitoring station on the ESP32 that configures the internal Pulse Counter (PCNT) hardware peripheral to count encoder pulses on GPIO registers, bypassing CPU interrupts.

## Goal
Learn how to configure the ESP32 hardware PCNT module, setup pulse and control signal filter limits, read 16-bit hardware counter registers, and monitor motor encoders.

## What You Will Build
An L298N driver controls a DC motor (IN1: 18, IN2: 19, ENA: 5). A quadrature encoder is connected to GPIO 4 (Pulse input) and GPIO 15 (Direction control input). A potentiometer on GPIO 34 controls motor speed and direction. The ESP32's hardware PCNT module counts the encoder pulses, letting the CPU read the count directly from registers.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motor with Encoder | `encoder_motor` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| External Power Supply (6-12V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 / IN2 / ENA | GPIO18 / GPIO19 / GPIO5 | Yellow/Green/Orange | Motor speed controls |
| Motor Encoder | OUT A (Pulse) | GPIO4 | Yellow | PCNT Pulse Input (GPIO 4) |
| Motor Encoder | OUT B (Dir) | GPIO15 | Green | PCNT Control Input (GPIO 15) |
| Potentiometer | Wiper | GPIO34 | Yellow | Motor speed control wiper |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the encoder outputs to GPIO 4 (Pulse) and GPIO 15 (Direction). Ground the encoder and L298N logic circuits to the ESP32 ground.

## Code
```cpp
// Hardware Pulse Counter (PCNT Module reads encoder)
#include <Arduino.h>
#include "driver/pcnt.h"

const int IN1 = 18;
const int IN2 = 19;
const int ENA_PIN = 5;
const int POT_PIN = 34;

// PCNT Hardware definitions
#define PCNT_INPUT_SIG_PIN   4  // Pulse signal pin
#define PCNT_INPUT_CTRL_PIN  15 // Direction control pin
#define PCNT_UNIT            PCNT_UNIT_0

// Track cumulative pulses
int16_t currentCount = 0;

void initPCNT() {
  // 1. Configure PCNT Unit Settings
  pcnt_config_t pcnt_config = {
    .pulse_gpio_num = PCNT_INPUT_SIG_PIN,
    .ctrl_gpio_num = PCNT_INPUT_CTRL_PIN,
    .lctrl_mode = PCNT_MODE_REVERSE,       // Reverse count direction if control pin is LOW
    .hctrl_mode = PCNT_MODE_KEEP,          // Keep count direction if control pin is HIGH
    .pos_mode = PCNT_COUNT_INC,            // Increment count on rising edge of signal
    .neg_mode = PCNT_COUNT_DIS,            // Ignore falling edges
    .counter_h_lim = 32767,                // Maximum positive limit (16-bit signed)
    .counter_l_lim = -32768,               // Minimum negative limit
    .unit = PCNT_UNIT,
    .channel = PCNT_CHANNEL_0
  };

  // 2. Initialize PCNT unit
  pcnt_unit_config(&pcnt_config);

  // 3. Configure glitch filter (ignore pulses shorter than 100 APB clock cycles = ~1.25 us)
  // Filters out high-frequency noise spikes
  pcnt_set_filter_value(PCNT_UNIT, 100);
  pcnt_filter_enable(PCNT_UNIT);

  // 4. Initialize and start the counter
  pcnt_counter_clear(PCNT_UNIT);
  pcnt_counter_resume(PCNT_UNIT);
}

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  
  // Set up ENA PWM using LEDC
  ledcSetup(0, 2000, 8);
  ledcAttachPin(ENA_PIN, 0);
  
  initPCNT();
  Serial.println("PCNT hardware counter configured.");
}

void loop() {
  // Read potentiometer and convert to motor speed and direction
  // pot = 2048 -> motor stopped
  // pot > 2048 -> motor forward (scale 0-255)
  // pot < 2048 -> motor reverse (scale 0-255)
  int potVal = analogRead(POT_PIN);
  int speed = 0;
  
  if (potVal > 2100) {
    speed = map(potVal, 2100, 4095, 0, 255);
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    ledcWrite(0, speed);
  } else if (potVal < 2000) {
    speed = map(potVal, 0, 2000, 255, 0);
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, HIGH);
    ledcWrite(0, speed);
  } else {
    // Stop motor
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    ledcWrite(0, 0);
  }
  
  // 5. Read the hardware counter register
  // Reads the value directly from the PCNT hardware registers
  pcnt_get_counter_value(PCNT_UNIT, &currentCount);
  
  Serial.print("Pot: "); Serial.print(potVal);
  Serial.print(" | Speed: "); Serial.print(speed);
  Serial.print(" | PCNT Register Count: "); Serial.println(currentCount);
  
  delay(200); // 5Hz log rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, **DC Motor with Encoder**, and **Potentiometer** onto the canvas.
2. Wire ENA to **GPIO5**, IN1/IN2 to **GPIO18/GPIO19**, Encoder outputs to **GPIO4/GPIO15**, and Potentiometer to **GPIO34**.
3. Paste the code and click **Run**.
4. Slide the potentiometer widget right. Watch the motor rotate forward and the PCNT register count increase.
5. Slide the potentiometer widget left. Watch the motor reverse and the PCNT register count decrease.

## Expected Output
Serial Monitor:
```
PCNT hardware counter configured.
Pot: 3200 | Speed: 140 | PCNT Register Count: 482
Pot: 3200 | Speed: 140 | PCNT Register Count: 1040
Pot: 1000 | Speed: 128 | PCNT Register Count: 820
```

## Expected Canvas Behavior
* At boot, the motor is stopped, and the counter reads 0.
* Sliding the potentiometer slider past 2100 spins the motor clockwise. The PCNT count increments.
* Sliding it below 2000 spins the motor counter-clockwise. The PCNT count decrements.
* The count changes smoothly in sync with the motor.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `lctrl_mode` | Configures the PCNT to reverse the count direction if the control pin is LOW. |
| `pcnt_set_filter_value(...)` | Configures the glitch filter to ignore high-frequency noise spikes on the input lines. |
| `pcnt_get_counter_value(...)` | Reads the 16-bit signed counter value directly from the hardware registers. |

## Hardware & Safety Concept: Hardware Pulse Counters vs Interrupt Overload
Quadrature encoders generate high-speed pulse streams (often exceeding 10 kHz). If you count these pulses using standard GPIO interrupts, the CPU must stop what it is doing to run the ISR 10,000 times per second, which consumes up to 80% of CPU time and causes lag. The ESP32 contains **8 hardware PCNT (Pulse Counter) units**. These units count pulses directly on registers in the background, freeing the CPU to perform other tasks and protecting against missed counts.

## Try This! (Challenges)
1. **Interactive OLED RPM display**: Add an OLED screen (Project 60) and display the motor's actual speed in RPM in real time.
2. **Interrupt on limit reached**: Configure a PCNT interrupt to trigger a callback when the counter reaches a target limit (e.g. 1000 pulses).
3. **Dual Motor Tracker**: Initialize a second PCNT unit (PCNT_UNIT_1) on GPIO 12 and 13 to track a second motor.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Counter does not count (reads static 0) | Signal lines miswired | Verify that the encoder's signal pin is connected to GPIO 4 |
| Counter increments but never decrements | Direction pin misconfigured | Check the control pin connection to GPIO 15 and verify the lctrl/hctrl modes |
| Count jumps randomly when still | Electrical noise | Enable the PCNT glitch filter and adjust the filter limit |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [151 - ESP32 DC Motor PID Speed Controller](151-esp32-dc-motor-pid-speed-controller.md)
- [181 - ESP32 Custom Hardware Timer Interrupts](181-esp32-custom-hardware-timer-interrupts.md)
- [123 - ESP32 Robot Speed Control](../intermediate/123-esp32-robot-speed-control.md)
