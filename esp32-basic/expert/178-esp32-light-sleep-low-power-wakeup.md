# 178 - ESP32 Light Sleep Low-power Wakeup

Build a low-power application on the ESP32 that demonstrates Light Sleep mode, enabling the CPU to suspend and resume program execution from the exact line following the sleep call, preserving variables and state.

## Goal
Learn how to configure light sleep states, contrast deep sleep vs. light sleep behavior, set up GPIO wakeup sources, and resume execution.

## What You Will Build
A Green LED is on GPIO 12 (Active indicator), a Red LED on GPIO 13 (Sleep indicator), and a push button on GPIO 4. The ESP32 blinks the Green LED, enters Light Sleep, and turns the Red LED ON. On a 5-second timer timeout or button press, the CPU wakes up, turns the Red LED OFF, and resumes execution from the next line, retaining all variable states.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistors (2) | `resistor` | No | Yes |
| 10 kΩ Resistor (1) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | Active status indicator |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Sleep status indicator |
| Push Button | Pin 1 / Pin 2 | GPIO4 / 3V3 | Orange / Red | Resume button (active-HIGH) |
| Push Button | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Common ground| GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the button with an external 10 kΩ pull-down resistor to GPIO 4. Standard GPIOs can be used as wakeup sources during Light Sleep.

## Code
```cpp
// Light Sleep Low-power Wakeup (GPIO Wakeup & Program Resume)
#include <Arduino.h>
#include <esp_sleep.h>
#include <driver/gpio.h>

const int LED_GREEN = 12;
const int LED_RED = 13;
const gpio_num_t BUTTON_GPIO = GPIO_NUM_4;

// Local variable to track cycles (retained during light sleep without RTC attributes)
int cycleCounter = 0;

void setup() {
  Serial.begin(115200);
  delay(500);
  
  pinMode(LED_GREEN, OUTPUT);
  pinMode(LED_RED, OUTPUT);
  
  digitalWrite(LED_GREEN, LOW);
  digitalWrite(LED_RED, LOW);
  
  // Configure button as input (external pull-down wired)
  pinMode(BUTTON_GPIO, INPUT);
  
  Serial.println("--- ESP32 Light Sleep Station Configured ---");
}

void loop() {
  cycleCounter++;
  
  Serial.print("\n=== STARTING ACTIVE CYCLE #"); Serial.print(cycleCounter); Serial.println(" ===");
  
  // Blink Green LED 3 times to show active mode
  for (int i = 0; i < 3; i++) {
    digitalWrite(LED_GREEN, HIGH); delay(100);
    digitalWrite(LED_GREEN, LOW);  delay(100);
  }
  
  // Configure wakeup sources for Light Sleep:
  // 1. Timer Wakeup: Wake up in 5 seconds (5,000,000 microseconds)
  esp_sleep_enable_timer_wakeup(5ULL * 1000000ULL);
  
  // 2. GPIO Wakeup: Wake up on GPIO 4 rising to HIGH level
  // Light sleep allows using the ESP32's internal GPIO wakeup hardware
  gpio_wakeup_enable(BUTTON_GPIO, GPIO_INTR_HIGH_LEVEL);
  esp_sleep_enable_gpio_wakeup();
  
  // Turn ON Red LED to indicate sleep mode
  digitalWrite(LED_RED, HIGH);
  
  Serial.println("Entering Light Sleep mode... CPU suspending.");
  delay(50); // Let Serial print finish
  
  // Enter Light Sleep
  esp_light_sleep_start();
  
  // ==========================================
  // WAKEUPS RESUME EXECUTION DIRECTLY HERE!
  // ==========================================
  
  // Turn OFF Red LED
  digitalWrite(LED_RED, LOW);
  
  Serial.println("CPU Resumed execution! Parsing wakeup trigger...");
  
  // Identify Wakeup Cause
  esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();
  if (cause == ESP_SLEEP_WAKEUP_TIMER) {
    Serial.println("Wakeup Source: RTC Timer expired.");
  } else if (cause == ESP_SLEEP_WAKEUP_GPIO) {
    Serial.println("Wakeup Source: Button pressed (GPIO 4).");
  } else {
    Serial.println("Wakeup Source: Unknown event.");
  }
  
  // Stay active for 2 seconds before sleeping again
  delay(2000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Button**, and two **LEDs** onto the canvas.
2. Wire Button to **GPIO4**, Green LED to **GPIO12**, and Red LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Watch the Green LED blink, and the Red LED turn ON. The system is in Light Sleep.
5. Wait 5 seconds. Watch the Red LED turn OFF, Green blink, and `cycleCounter` increment.
6. Click the button widget during sleep. The system resumes instantly.

## Expected Output
Serial Monitor:
```
--- ESP32 Light Sleep Station Configured ---

=== STARTING ACTIVE CYCLE #1 ===
Entering Light Sleep mode... CPU suspending.
CPU Resumed execution! Parsing wakeup trigger...
Wakeup Source: RTC Timer expired.

=== STARTING ACTIVE CYCLE #2 ===
Entering Light Sleep mode... CPU suspending.
CPU Resumed execution! Parsing wakeup trigger...
Wakeup Source: Button pressed (GPIO 4).
```

## Expected Canvas Behavior
* At boot, the Green LED blinks.
* When entering sleep, the Red LED turns ON.
* After 5 seconds of inactivity, the Red LED turns OFF and the Green LED blinks again.
* If you click the button widget while the Red LED is ON, the Red LED turns OFF instantly and the Green LED blinks, indicating the system has resumed.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `cycleCounter` | Local variable that retains its value across cycles because RAM is powered during Light Sleep. |
| `gpio_wakeup_enable(...)` | Binds any standard GPIO pin to function as a wakeup source. |
| `esp_light_sleep_start()` | Suspends CPU execution and puts peripherals into a low-power state. |
| `digitalWrite(LED_RED, LOW)` | Resumes execution from the next line after waking up. |

## Hardware & Safety Concept: Light Sleep vs. Deep Sleep
The ESP32 offers two low-power sleep modes:
1. **Deep Sleep**: Powers down the CPU, RAM, and most registers. The CPU resets and runs `setup()` on wakeup. It draws minimal current (**10–15 µA**), but loses variable states.
2. **Light Sleep**: Suspends the CPU and clocks, but keeps RAM powered. The CPU resumes execution from the next line on wakeup, preserving variable states. It draws slightly more current (**0.8–2.0 mA**), but allows fast resume.

## Try This! (Challenges)
1. **Interactive blink speed**: Increase the Green LED blink count dynamically on button presses.
2. **Serial command wakeup**: Configure the UART interface to wake the ESP32 when serial characters are received.
3. **Dual sleep toggle**: Add a second button on GPIO 14 that toggles the system between Light Sleep and Deep Sleep.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| System resets and starts from setup on wakeup | System entered Deep Sleep instead of Light Sleep | Verify that `esp_light_sleep_start()` is called, not `esp_deep_sleep_start()` |
| Button does not wake the system | GPIO wakeup not enabled | Ensure both `gpio_wakeup_enable` and `esp_sleep_enable_gpio_wakeup` are called |
| Sleep current is high in hardware | Connected pins leaking current | Configure unused GPIOs as inputs with pull-ups to prevent leakage |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [177 - ESP32 Deep Sleep Low-power Wakeup](177-esp32-deep-sleep-low-power-wakeup.md)
- [36 - ESP32 PIR Motion Sensor Alert](../beginner/36-esp32-pir-motion-sensor-alert.md)
- [171 - ESP32 Multi-tasking RTOS Scheduler Demo](171-esp32-multi-tasking-rtos-scheduler-demo.md)
