# 181 - ESP32 Custom Hardware Timer Interrupts

Build a high-precision pulse generator on the ESP32 that configures an internal hardware timer interrupt to toggle a status pin, and uses a potentiometer to dynamically adjust the interrupt frequency in microseconds.

## Goal
Learn how to configure the ESP32's hardware timers, register high-speed interrupt service routines (ISRs), and dynamically modify timer thresholds.

## What You Will Build
A Green LED is on GPIO 12, a Red LED on GPIO 13, and a potentiometer on GPIO 34. Hardware Timer 0 is configured to generate ticks at 1 MHz (1 tick = 1 microsecond). The timer triggers an interrupt to toggle the Green LED. A potentiometer on GPIO 34 dynamically adjusts the trigger period (from 50 ms to 1 second). The Red LED blinks in the background to show non-blocking operation.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| 330 Ω Resistors (2) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | Timer ISR output target |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Background loop indicator |
| Potentiometer | Wiper | GPIO34 | Yellow | Frequency control wiper |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the common ground rail for the LED cathodes and potentiometer ground. Power the potentiometer from the 3.3V pin.

## Code
```cpp
// Custom Hardware Timer Interrupts (Precise Timer ISR)
#include <Arduino.h>
#include "soc/gpio_reg.h"

const int LED_GREEN = 12;
const int LED_RED = 13;
const int POT_PIN = 34;

// Hardware Timer pointer
hw_timer_t * timer = NULL;

// Volatile variable changed inside ISR
volatile bool ledState = false;

// Interrupt Service Routine (ISR) called on timer alarm
// Marked with IRAM_ATTR to run inside fast RAM
void IRAM_ATTR onTimerISR() {
  ledState = !ledState;
  
  // Toggle Green LED (GPIO 12) instantly using fast direct register writes
  if (ledState) {
    REG_WRITE(GPIO_OUT_W1TS_REG, (1 << 12));
  } else {
    REG_WRITE(GPIO_OUT_W1TC_REG, (1 << 12));
  }
}

void setup() {
  Serial.begin(115200);
  delay(500);
  
  pinMode(LED_GREEN, OUTPUT);
  pinMode(LED_RED, OUTPUT);
  pinMode(POT_PIN, INPUT);
  
  digitalWrite(LED_GREEN, LOW);
  digitalWrite(LED_RED, LOW);
  
  // 1. Initialize Hardware Timer 0
  // 80: Clock prescaler. Since the ESP32 base clock is 80 MHz,
  // dividing by 80 yields a 1 MHz tick rate (1 tick = 1 microsecond).
  // true: Count upwards
  timer = timerBegin(0, 80, true); 
  
  // 2. Attach interrupt handler function to the timer
  timerAttachInterrupt(timer, &onTimerISR, true);
  
  // 3. Configure initial alarm trigger threshold
  // 500000 ticks = 500,000 microseconds = 500 ms period
  // true: Auto-reload the timer to repeat the alarm continuously
  timerAlarmWrite(timer, 500000, true); 
  
  // 4. Enable the alarm
  timerAlarmEnable(timer);
  
  Serial.println("Hardware Timer 0 initialized at 1 MHz tick rate.");
}

void loop() {
  // Read potentiometer and map to alarm period in microseconds (50ms to 1s)
  int potRaw = analogRead(POT_PIN);
  uint64_t alarmPeriodUs = map(potRaw, 0, 4095, 50000, 1000000); // 50,000 to 1,000,000 us
  
  // 5. Dynamically update the timer alarm threshold
  // This adjusts the toggle frequency in the background
  timerAlarmWrite(timer, alarmPeriodUs, true);
  
  // Calculate output frequency
  float frequencyHz = 1000000.0 / (float)(alarmPeriodUs * 2);
  
  Serial.print("Pot Raw: "); Serial.print(potRaw);
  Serial.print(" | Period: "); Serial.print(alarmPeriodUs / 1000.0); Serial.print(" ms");
  Serial.print(" | Freq: "); Serial.print(frequencyHz, 2); Serial.println(" Hz");
  
  // 6. Blink Red LED slowly in the background
  // Demonstrates that the timer interrupt runs independently in the background
  digitalWrite(LED_RED, HIGH);
  delay(100);
  digitalWrite(LED_RED, LOW);
  
  delay(900); // Main loop runs once per second
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer**, and two **LEDs** onto the canvas.
2. Wire Potentiometer to **GPIO34**, Green LED to **GPIO12**, and Red LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Watch the Red LED blink slowly once per second.
5. Slide the potentiometer widget. Watch the Green LED toggle speed change dynamically, running completely in the background.

## Expected Output
Serial Monitor:
```
Hardware Timer 0 initialized at 1 MHz tick rate.
Pot Raw: 2048 | Period: 525.0 ms | Freq: 0.95 Hz
Pot Raw: 4095 | Period: 1000.0 ms | Freq: 0.50 Hz
Pot Raw: 0 | Period: 50.0 ms | Freq: 10.00 Hz
```

## Expected Canvas Behavior
* The Red LED flashes once per second (100 ms ON, 900 ms OFF).
* The Green LED flashes at a speed set by the potentiometer.
* Sliding the potentiometer slider all the way left causes the Green LED to flash rapidly (10 Hz).
* Sliding it all the way right slows the Green LED to a 1 Hz flash rate.

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `timerBegin(0, 80, true)` | Configures Hardware Timer 0 with an 80-prescaler to establish a 1-microsecond base tick rate. |
| `timerAttachInterrupt(...)` | Links the timer to trigger the registered ISR function. |
| `timerAlarmWrite(...)` | Sets the target alarm period in ticks. Auto-reload is enabled to repeat the alarm. |
| `REG_WRITE(...)` | Writes to the GPIO registers inside the ISR to toggle the pin instantly. |

## Hardware & Safety Concept: Hardware Timers vs Software Delays
Software delays (like `delay()` or `vTaskDelay()`) rely on the CPU executing clock cycles or context switching, which can be interrupted by other tasks. For high-precision applications (like generating a 38 kHz carrier wave for an IR remote or driving stepper motor pulses), software delays are too jittery. **Hardware Timers** run independently of the CPU core. Once the counter reaches the set threshold, it interrupts the CPU instantly, running the registered ISR with sub-microsecond precision.

## Try This! (Challenges)
1. **Pulse Width Modulation (PWM) Generator**: Modify the timer to alternate between two different periods (e.g. 100 us ON, 900 us OFF) to create a custom software PWM signal.
2. **Frequency Sweeper**: Automatically sweep the timer frequency up and down to create a siren tone on a buzzer (GPIO 12).
3. **Timer Stop button**: Add a button on GPIO 4 that disables the timer alarm when pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| System crashes immediately on boot | Too much work in the ISR | Keep the ISR code as short as possible; only toggle pins or write to registers inside the ISR |
| Green LED does not blink | Alarm not enabled | Ensure `timerAlarmEnable()` is called in `setup()` |
| Toggle frequency is twice as fast as expected | Period configuration error | Remember that toggling a pin HIGH and LOW requires two interrupts per full square wave cycle |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [179 - ESP32 Direct Register Port Manipulation](179-esp32-direct-register-port-manipulation.md)
- [175 - ESP32 RTOS Task Notifications Alarm](175-esp32-task-notifications-alarm.md)
- [53 - ESP32 Variable LED Blink Rates](../intermediate/53-esp32-variable-led-blink-rates.md)
