# 175 - ESP32 RTOS Task Notifications Alarm

Build a high-performance RTOS interrupt handler that binds a hardware button interrupt service routine (ISR) directly to a high-priority alarm task using FreeRTOS Task Notifications, sounding a buzzer instantly when triggered.

## Goal
Learn how to use FreeRTOS Task Notifications to send fast, lightweight signals from a hardware ISR to a task, and understand context switching (`portYIELD_FROM_ISR`).

## What You Will Build
A push button is connected to GPIO 4, and an active buzzer is on GPIO 13. When the button is pressed, it triggers a hardware interrupt. The ISR immediately sends a FreeRTOS Task Notification to a sleeping, high-priority Alarm Task. The task wakes up instantly and sounds the buzzer for 500 ms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Pin 1 / Pin 2 | GPIO4 / 3V3 | Orange / Red | Emergency ISR input (active-HIGH) |
| Push Button | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Active Buzzer | VCC (+) | GPIO13 | Blue | Alarm output |
| Common ground| GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the button with an external 10 kΩ pull-down resistor to GPIO 4. In hardware, this prevents electrical noise from falsely triggering the interrupt.

## Code
```cpp
// RTOS Task Notifications Alarm (ISR Warning Alerts)
#include <Arduino.h>

const int BUTTON_PIN = 4;
const int BUZZER_PIN = 13;

// Task Handle for the alarm coordinator
TaskHandle_t AlarmTaskHandle = NULL;

// Hardware Interrupt Service Routine (ISR)
// Marked with IRAM_ATTR to run inside the fast RAM block
void IRAM_ATTR emergencyISR() {
  BaseType_t xHigherPriorityTaskWoken = pdFALSE;
  
  // Send notification directly to the alarm task
  vTaskNotifyGiveFromISR(AlarmTaskHandle, &xHigherPriorityTaskWoken);
  
  // Force a context switch if the notified task has a higher priority than the current task
  portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

// Alarm Task: Sleeps until notified by the ISR
void TaskAlarmProcessor(void *pvParameters) {
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  while (1) {
    // Block and wait indefinitely for a notification
    // pdTRUE: resets the notification value back to 0 on exit
    uint32_t threadVal = ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
    
    if (threadVal > 0) {
      Serial.println("[Alarm Task] emergency notification received!");
      
      // Sound alarm
      digitalWrite(BUZZER_PIN, HIGH);
      vTaskDelay(pdMS_TO_TICKS(500)); // Beep for 500 ms
      digitalWrite(BUZZER_PIN, LOW);
    }
  }
}

// Low-priority Idle Task: Simulates background CPU activity
void TaskBackgroundWork(void *pvParameters) {
  while (1) {
    Serial.println("Background task executing CPU operations...");
    vTaskDelay(pdMS_TO_TICKS(2000));
  }
}

void setup() {
  Serial.begin(115200);
  Serial.println("Initializing Task Notification demo...");
  
  // Create the high-priority Alarm Task (Priority 3)
  xTaskCreate(
    TaskAlarmProcessor,
    "Alarm Processor",
    2048,
    NULL,
    3, // High priority
    &AlarmTaskHandle
  );
  
  // Create background task (Priority 1)
  xTaskCreate(
    TaskBackgroundWork,
    "Background Work",
    1024,
    NULL,
    1, // Low priority
    NULL
  );
  
  // Attach hardware interrupt to the button pin (rising edge)
  pinMode(BUTTON_PIN, INPUT);
  attachInterrupt(digitalPinToInterrupt(BUTTON_PIN), emergencyISR, RISING);
  
  Serial.println("Interrupts and tasks online.");
}

void loop() {
  vTaskDelay(pdMS_TO_TICKS(1000));
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Button**, and **Buzzer** onto the canvas.
2. Wire Button to **GPIO4** and Buzzer to **GPIO13**.
3. Paste the code and click **Run**.
4. Monitor the Serial Monitor. Watch the background task log messages.
5. Click the button widget. Watch the buzzer beep instantly, interrupting the background logs.

## Expected Output
Serial Monitor:
```
Initializing Task Notification demo...
Interrupts and tasks online.
Background task executing CPU operations...
[Alarm Task] emergency notification received!
Background task executing CPU operations...
```

## Expected Canvas Behavior
* The background task prints logs every 2 seconds.
* Clicking the button widget instantly triggers the buzzer widget to pulse green for 500 ms.
* The alarm task responds immediately without any lag.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `vTaskNotifyGiveFromISR(...)` | Sends a fast notification event from the hardware ISR directly to the specified task. |
| `portYIELD_FROM_ISR(...)` | Instructs the scheduler to swap task contexts immediately, waking up the alarm task without waiting for the current time-slice to end. |
| `ulTaskNotifyTake(...)` | Blocks the alarm task until a notification is received, entering a low-power sleep state. |

## Hardware & Safety Concept: Task Notifications vs Binary Semaphores
In real-time operating systems (RTOS), hardware interrupts must be handled as quickly as possible. If an ISR does too much work (like printing to serial or sleeping), it blocks the CPU, destabilizing the system. Instead, the ISR should only send a signal to a helper task. While **Binary Semaphores** can do this, **Task Notifications** are up to 45% faster and use less RAM because they send the signal directly to the task's TCB (Task Control Block) instead of using an intermediate queue.

## Try This! (Challenges)
1. **Multi-Press Alarm Tracker**: Modify `ulTaskNotifyTake` to check the notification count, beep multiple times if the button was pressed multiple times during a delay.
2. **Emergency LED warning**: Flash a Red LED on GPIO 12 while the buzzer is active.
3. **Emergency Reset combination**: Add a second button that requires a specific press pattern to clear the alarm task state.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Microcontroller crashes when button is pressed | ISR calling unsafe functions | Never call blocking functions (like `delay` or `Serial.print`) inside an ISR. Only use safe ISR functions |
| Alarm task does not run | Task handle not initialized | Ensure the task handle pointer is passed correctly to `xTaskCreate` and is not NULL when the ISR runs |
| Double trigger on a single press | Button bounce | Install a hardware capacitor (0.1 µF) across the button to filter out noise |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [171 - ESP32 Multi-tasking RTOS Scheduler Demo](171-esp32-multi-tasking-rtos-scheduler-demo.md)
- [174 - ESP32 RTOS Event Groups System](174-esp32-rtos-event-groups-system.md)
- [176 - ESP32 Hardware Watchdog Timer Recovery](176-esp32-hardware-watchdog-timer-recovery.md) (Next project)
