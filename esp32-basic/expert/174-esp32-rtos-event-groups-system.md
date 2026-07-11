# 174 - ESP32 RTOS Event Groups System

Build an RTOS application on the ESP32 that uses a FreeRTOS Event Group to coordinate a high-priority buzzer controller task, waking it up only when two independent button monitoring tasks have both set their respective event flags (AND condition).

## Goal
Learn how to create FreeRTOS Event Groups, use event flags (bits) to communicate states, and synchronize tasks using complex combinations of flags (AND/OR conditions).

## What You Will Build
Two push buttons are connected to GPIO 4 (Button A) and GPIO 12 (Button B). An active buzzer is on GPIO 13. Two independent tasks monitor the buttons and set event flags when they are pressed. A third coordinator task waits for **both** flags to be set (Button A AND Button B pressed), then sounds the buzzer for 1 second and clears the flags.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Push Buttons (2) | `button` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 10 kΩ Resistors (2) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Button A | Pin 1 / Pin 2 | GPIO4 / 3V3 | Orange / Red | Monitor Task A input (active-HIGH) |
| Button A | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Button B | Pin 1 / Pin 2 | GPIO12 / 3V3 | Green / Red | Monitor Task B input (active-HIGH) |
| Button B | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Active Buzzer | VCC (+) | GPIO13 | Blue | Alarm buzzer |
| Common ground| GND | GND | Black | Shared reference |

> **Wiring tip:** Connect both buttons with external 10 kΩ pull-down resistors to their respective GPIO pins (4 and 12) to ensure clean digital transitions.

## Code
```cpp
// RTOS Event Groups System (Button Flags Coordination)
#include <Arduino.h>

const int BUTTON_A_PIN = 4;
const int BUTTON_B_PIN = 12;
const int BUZZER_PIN = 13;

// Event Group Handle
EventGroupHandle_t sensorEventGroup;

// Define Event Bits (Flags)
const EventBits_t BIT_BUTTON_A = (1 << 0); // Flag for Button A
const EventBits_t BIT_BUTTON_B = (1 << 1); // Flag for Button B
const EventBits_t ALL_BITS = (BIT_BUTTON_A | BIT_BUTTON_B);

// Task 1: Monitor Button A and set flag
void TaskButtonA(void *pvParameters) {
  pinMode(BUTTON_A_PIN, INPUT);
  bool lastState = LOW;
  
  while (1) {
    bool state = (digitalRead(BUTTON_A_PIN) == HIGH);
    if (state != lastState) {
      lastState = state;
      if (state == HIGH) {
        Serial.println("[Task A] Button A pressed. Setting Flag A.");
        // Set flag bit
        xEventGroupSetBits(sensorEventGroup, BIT_BUTTON_A); 
      }
    }
    vTaskDelay(pdMS_TO_TICKS(50)); // Poll every 50 ms
  }
}

// Task 2: Monitor Button B and set flag
void TaskButtonB(void *pvParameters) {
  pinMode(BUTTON_B_PIN, INPUT);
  bool lastState = LOW;
  
  while (1) {
    bool state = (digitalRead(BUTTON_B_PIN) == HIGH);
    if (state != lastState) {
      lastState = state;
      if (state == HIGH) {
        Serial.println("[Task B] Button B pressed. Setting Flag B.");
        // Set flag bit
        xEventGroupSetBits(sensorEventGroup, BIT_BUTTON_B); 
      }
    }
    vTaskDelay(pdMS_TO_TICKS(50)); // Poll every 50 ms
  }
}

// Task 3: Coordinator - Waits for BOTH flags to be set, then sounds buzzer
void TaskBuzzerCoordinator(void *pvParameters) {
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  while (1) {
    Serial.println("[Coordinator] Waiting for both flags A AND B...");
    
    // Wait for bits to be set:
    // 1. Event group handle
    // 2. Bits to wait for
    // 3. Clear on exit (pdTRUE: clear flags once condition is met)
    // 4. Wait for all bits (pdTRUE: AND condition; pdFALSE: OR condition)
    // 5. Timeout (portMAX_DELAY: wait indefinitely)
    EventBits_t uxBits = xEventGroupWaitBits(
      sensorEventGroup,
      ALL_BITS,
      pdTRUE, // Clear bits on exit
      pdTRUE, // Wait for BOTH bits (AND)
      portMAX_DELAY
    );
    
    // Once both flags are set, the task resumes here
    if ((uxBits & ALL_BITS) == ALL_BITS) {
      Serial.println("[Coordinator] !!! BOTH FLAGS SET: Activating Buzzer !!!");
      digitalWrite(BUZZER_PIN, HIGH);
      vTaskDelay(pdMS_TO_TICKS(1000)); // Beep for 1 second
      digitalWrite(BUZZER_PIN, LOW);
    }
  }
}

void setup() {
  Serial.begin(115200);
  Serial.println("Initializing Event Group...");
  
  // Create Event Group
  sensorEventGroup = xEventGroupCreate();
  
  if (sensorEventGroup != NULL) {
    Serial.println("Event Group created successfully.");
    
    // Create Tasks
    xTaskCreate(TaskButtonA, "Task Button A", 2048, NULL, 1, NULL);
    xTaskCreate(TaskButtonB, "Task Button B", 2048, NULL, 1, NULL);
    xTaskCreate(TaskBuzzerCoordinator, "Buzzer Coord", 2048, NULL, 2, NULL); // Higher priority
  } else {
    Serial.println("Error creating Event Group!");
    while(1) {}
  }
}

void loop() {
  vTaskDelay(pdMS_TO_TICKS(1000));
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, two **Buttons**, and a **Buzzer** onto the canvas.
2. Wire Button A to **GPIO4**, Button B to **GPIO12**, and Buzzer to **GPIO13**.
3. Paste the code and click **Run**.
4. Click Button A. Watch the Serial Monitor print `Setting Flag A`. The buzzer does not sound.
5. Click Button B. Watch the buzzer beep for 1 second now that both flags are set.

## Expected Output
Serial Monitor:
```
Initializing Event Group...
Event Group created successfully.
[Coordinator] Waiting for both flags A AND B...
[Task A] Button A pressed. Setting Flag A.
[Task B] Button B pressed. Setting Flag B.
[Coordinator] !!! BOTH FLAGS SET: Activating Buzzer !!!
[Coordinator] Waiting for both flags A AND B...
```

## Expected Canvas Behavior
* Clicking only one button does not sound the buzzer.
* Once both buttons have been clicked, the buzzer widget pulses green for 1 second.
* After sounding, the flags are cleared, and the coordinator task waits for both buttons to be pressed again.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `xEventGroupCreate()` | Allocates memory for a FreeRTOS event group holding up to 24 flags. |
| `xEventGroupSetBits(...)` | Sets specific bit flags in the event group, notifying waiting tasks. |
| `xEventGroupWaitBits(...)` | Blocks the coordinator task until the specified flags are set. |

## Hardware & Safety Concept: Synchronization and Event Groups
In complex embedded systems, actions are often triggered by a combination of multiple conditions (e.g. pressure high AND temperature high AND safety gate closed). Using global variables to track these flags requires continuous polling, which wastes CPU time. FreeRTOS **Event Groups** allow tasks to block and wait for multiple conditions to be met, sleeping until the scheduler wakes them up, saving power.

## Try This! (Challenges)
1. **OR Trigger Mode**: Change `xEventGroupWaitBits` to wake up if **either** button is pressed (OR condition) and sound the buzzer.
2. **Timeout Reset alert**: Set a timeout of 5 seconds. If the user presses Button A but does not press Button B within 5 seconds, clear the flags and flash an error.
3. **Emergency Stop Override**: Add a third button (GPIO 14) that sets a high-priority flag, disabling all tasks instantly.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound when both buttons are pressed | Flag bits not cleared on exit | Verify that the `xClearOnExit` parameter is set to `pdTRUE` in `xEventGroupWaitBits` |
| Coordinator task runs continuously | `xEventGroupWaitBits` parameter error | Ensure the timeout is set to `portMAX_DELAY` to block the task correctly |
| Button triggers multiple flags | Button bounce | Add debounce delay or poll checks to filter out noise |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [171 - ESP32 Multi-tasking RTOS Scheduler Demo](171-esp32-multi-tasking-rtos-scheduler-demo.md)
- [173 - ESP32 RTOS Queue Data Producer-Consumer](173-esp32-rtos-queue-data-producer-consumer.md)
- [175 - ESP32 RTOS Task Notifications Alarm](175-esp32-rtos-task-notifications-alarm.md) (Next project)
