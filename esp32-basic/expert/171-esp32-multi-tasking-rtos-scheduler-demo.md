# 171 - ESP32 Multi-tasking RTOS Scheduler Demo

Build an RTOS application on the ESP32 that runs three concurrent, independent tasks (blinking a Green LED, blinking a Red LED, and polling a push button) managed by the built-in FreeRTOS preemptive scheduler.

## Goal
Learn how to create FreeRTOS tasks, assign priorities, manage task execution states, and use non-blocking RTOS delays (`vTaskDelay`) instead of `delay()`.

## What You Will Build
A Green LED is on GPIO 12, a Red LED on GPIO 13, and a push button on GPIO 4. The ESP32 does not use the standard Arduino single-loop execution model. Instead, it creates three separate FreeRTOS tasks that run in parallel. The preemptive scheduler swaps execution contexts to run all three tasks concurrently.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| 330 Ω Resistors (2) | `resistor` | No | Yes |
| 10 kΩ Resistor (1) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | Blink Task 1 target |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Blink Task 2 target |
| Push Button | Pin 1 / Pin 2 | GPIO4 / 3V3 | Orange / Red | Button poll input (active-HIGH) |
| Push Button | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Common ground| GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the push button with an external 10 kΩ pull-down resistor to GPIO 4 to prevent the input from floating when open.

## Code
```cpp
// Multi-tasking RTOS Scheduler Demo (FreeRTOS Tasks)
#include <Arduino.h>

const int LED_GREEN = 12;
const int LED_RED = 13;
const int BUTTON_PIN = 4;

// Task Handlers (optional, used to control or delete tasks)
TaskHandle_t GreenTaskHandle = NULL;
TaskHandle_t RedTaskHandle = NULL;
TaskHandle_t ButtonTaskHandle = NULL;

// Task 1: Blink Green LED every 500 ms
void TaskGreenBlink(void *pvParameters) {
  pinMode(LED_GREEN, OUTPUT);
  
  while (1) {
    digitalWrite(LED_GREEN, HIGH);
    // Non-blocking delay: block task for 500ms, letting CPU run other tasks
    vTaskDelay(pdMS_TO_TICKS(500)); 
    digitalWrite(LED_GREEN, LOW);
    vTaskDelay(pdMS_TO_TICKS(500));
  }
}

// Task 2: Blink Red LED every 1000 ms
void TaskRedBlink(void *pvParameters) {
  pinMode(LED_RED, OUTPUT);
  
  while (1) {
    digitalWrite(LED_RED, HIGH);
    vTaskDelay(pdMS_TO_TICKS(1000));
    digitalWrite(LED_RED, LOW);
    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

// Task 3: Read Push Button and log state changes
void TaskButtonPoll(void *pvParameters) {
  pinMode(BUTTON_PIN, INPUT);
  
  bool lastState = LOW;
  
  while (1) {
    bool currentState = (digitalRead(BUTTON_PIN) == HIGH);
    
    if (currentState != lastState) {
      lastState = currentState;
      if (currentState == HIGH) {
        Serial.println("[Button Task] Button PRESSED");
      } else {
        Serial.println("[Button Task] Button RELEASED");
      }
    }
    
    // Poll button state every 50 ms
    vTaskDelay(pdMS_TO_TICKS(50)); 
  }
}

void setup() {
  Serial.begin(115200);
  Serial.println("Initializing FreeRTOS Scheduler...");
  
  // Create tasks using xTaskCreate:
  // 1. Task function pointer
  // 2. Descriptive name for debugging
  // 3. Stack size in bytes (1024-2048 is standard)
  // 4. Parameter passed to task (NULL here)
  // 5. Task priority (higher number = higher priority, e.g. 1-24)
  // 6. Task handle pointer (optional)
  
  xTaskCreate(
    TaskGreenBlink,
    "Green Blink",
    1024,
    NULL,
    1, // Low priority
    &GreenTaskHandle
  );
  
  xTaskCreate(
    TaskRedBlink,
    "Red Blink",
    1024,
    NULL,
    1, // Low priority
    &RedTaskHandle
  );
  
  xTaskCreate(
    TaskButtonPoll,
    "Button Poll",
    2048, // Larger stack for serial printing functions
    NULL,
    2, // Higher priority to ensure fast response to user inputs
    &ButtonTaskHandle
  );
  
  Serial.println("FreeRTOS scheduler started. Running tasks.");
}

void loop() {
  // In an RTOS application, the Arduino loop task runs on Core 1 at priority 1.
  // We leave loop() empty or delete it to let the scheduler manage tasks.
  vTaskDelay(pdMS_TO_TICKS(1000));
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, two **LEDs**, and a **Button** onto the canvas.
2. Wire Green LED to **GPIO12**, Red LED to **GPIO13**, and Button to **GPIO4**.
3. Paste the code and click **Run**.
4. Watch the Green and Red LEDs blink independently.
5. Click the button widget. Watch the button press/release logs display instantly in the Serial Monitor.

## Expected Output
Serial Monitor:
```
Initializing FreeRTOS Scheduler...
FreeRTOS scheduler started. Running tasks.
[Button Task] Button PRESSED
[Button Task] Button RELEASED
```

## Expected Canvas Behavior
* At startup, the Green LED blinks every 500 ms and the Red LED blinks every 1000 ms.
* The blink cycles of the LEDs are independent and do not block each other.
* Clicking the button widget prints messages immediately without interfering with the LED blinks.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `vTaskDelay(pdMS_TO_TICKS(500))` | Blocks the current task for 500 ms, releasing CPU time to run other active tasks. |
| `xTaskCreate(...)` | Registers a new task with the FreeRTOS scheduler, allocating stack space and setting its priority. |
| `priority = 2` | Assigns the button polling task a higher priority to ensure fast response to inputs. |

## Hardware & Safety Concept: Preemptive Scheduling
Traditional microcontrollers run a single program loop. If one function takes a long time (like calling `delay(1000)`), all other tasks are blocked. The ESP32 runs **FreeRTOS**, a real-time operating system. The scheduler uses **preemptive time-slicing** to swap task execution contexts at regular intervals (typically every 1 ms). If a higher-priority task needs to run, the scheduler interrupts the lower-priority task instantly to ensure real-time responsiveness.

## Try This! (Challenges)
1. **Interactive Task Suspender**: Connect a second button on GPIO 14. When pressed, suspend the Red LED task using `vTaskSuspend(RedTaskHandle)`. When pressed again, resume it using `vTaskResume(RedTaskHandle)`.
2. **CPU Core Binder**: Use `xTaskCreatePinnedToCore` to run the blinking tasks on Core 0 and the button task on Core 1 to demonstrate multi-core processing.
3. **Emergency Stop Interrupt**: Create a high-priority task (Priority 5) that monitors a stop button, shutting down all tasks if pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| ESP32 crashes and prints stack overflow errors | Task stack size too small | Increase stack size in `xTaskCreate` (e.g. from 1024 to 2048 bytes) |
| Button presses are missed or delayed | Polling delay too long or task priority too low | Decrease the button task's poll delay to 10 ms or increase its priority |
| One LED stops blinking | Task entered an infinite loop without calling `vTaskDelay` | Ensure every task loop contains a blocking call like `vTaskDelay` to release CPU time |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [172 - ESP32 RTOS Mutex Shared Resource Protector](172-esp32-rtos-mutex-shared-resource-protector.md) (Next project)
- [183 - ESP32 Dual-core Processing Balancer](183-esp32-dual-core-processing-balancer.md)
- [56 - ESP32 Button Controlled Relay Switch](../intermediate/56-esp32-button-controlled-relay-switch.md)
