# 176 - ESP32 Hardware Watchdog Timer Recovery

Build a fault-tolerant safety system that initializes the ESP32 hardware Watchdog Timer (WDT), simulates a CPU lockup when a push button is pressed, and uses the WDT to trigger a hardware reset, logging the recovery status on boot.

## Goal
Learn how to initialize the Task Watchdog Timer (TWDT), check boot reset reasons, feed watchdogs to confirm normal operation, and recover from code lockups.

## What You Will Build
A Green LED is on GPIO 12, a Red LED on GPIO 13, and a push button on GPIO 4. At boot, the ESP32 checks if the reset was caused by a watchdog timeout. In normal operation, the Green LED flashes once per second, and the code resets the watchdog timer. If the button is pressed, the code enters an infinite loop (simulating a freeze). After 3 seconds, the watchdog resets the CPU, restoring normal operation.

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
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | Normal status indicator |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Simulated lockup indicator |
| Push Button | Pin 1 / Pin 2 | GPIO4 / 3V3 | Orange / Red | Freeze trigger input (active-HIGH) |
| Push Button | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Common ground| GND | GND | Black | Shared reference |

> **Wiring tip:** Ground the LED cathodes and the button pull-down resistor. Wire the button directly to GPIO 4.

## Code
```cpp
// Hardware Watchdog Timer Recovery (WDT Reset & Recovery)
#include <Arduino.h>
#include <esp_task_wdt.h>
#include <esp_system.h>

const int LED_GREEN = 12;
const int LED_RED = 13;
const int BUTTON_PIN = 4;

// Watchdog timeout window in seconds
const int WDT_TIMEOUT_SECONDS = 3; 

void checkResetReason() {
  esp_reset_reason_t reason = esp_reset_reason();
  
  Serial.print("Bootup Reset Reason Code: "); Serial.print(reason);
  
  switch (reason) {
    case ESP_RST_POWERON:
      Serial.println(" (Power-on Reset - normal boot)");
      break;
    case ESP_RST_TASK_WDT:
      Serial.println(" (!!! TASK WATCHDOG RESET - RECOVERY SUCCESSFUL !!!)");
      // Flash Red LED 5 times to show recovery warning
      for (int i = 0; i < 5; i++) {
        digitalWrite(LED_RED, HIGH); delay(100);
        digitalWrite(LED_RED, LOW);  delay(100);
      }
      break;
    case ESP_RST_SW:
      Serial.println(" (Software Reset)");
      break;
    default:
      Serial.println(" (Other Reset Reason)");
      break;
  }
}

void setup() {
  Serial.begin(115200);
  delay(500); // Let Serial monitor connect
  
  pinMode(LED_GREEN, OUTPUT);
  pinMode(LED_RED, OUTPUT);
  pinMode(BUTTON_PIN, INPUT); // External pull-down wired
  
  digitalWrite(LED_GREEN, LOW);
  digitalWrite(LED_RED, LOW);
  
  Serial.println("\n--- Watchdog Safety Station Booting ---");
  checkResetReason();
  
  // 1. Initialize Task Watchdog Timer (TWDT)
  // WDT_TIMEOUT_SECONDS: Timeout period in seconds
  // true: Trigger a hard CPU panic reset on timeout
  Serial.println("Initializing TWDT...");
  esp_task_wdt_init(WDT_TIMEOUT_SECONDS, true); 
  
  // 2. Subscribe the current loop task (main loop) to TWDT monitoring
  esp_task_wdt_add(NULL); 
  
  Serial.println("Watchdog monitoring active. Feeding loop task.");
}

void loop() {
  // 3. Monitor lockup trigger button
  bool triggerFreeze = (digitalRead(BUTTON_PIN) == HIGH);
  
  if (triggerFreeze) {
    Serial.println("\n!!! FREEZE TRIGGERED: Simulating Infinite Loop Lockup !!!");
    Serial.println("Stopping Watchdog feeds. System hanging...");
    
    // Light up Red LED to indicate lockup state
    digitalWrite(LED_GREEN, LOW);
    digitalWrite(LED_RED, HIGH);
    
    // Simulate lockup: infinite loop that never feeds the watchdog
    while (1) {
      // Loop forever, blocking CPU execution
    }
  }
  
  // 4. Normal Operation Code
  // Flash Green LED to indicate running normally
  digitalWrite(LED_GREEN, HIGH);
  delay(100);
  digitalWrite(LED_GREEN, LOW);
  
  // 5. Feed (reset) the Watchdog Timer
  // Confirms the loop task is running normally
  esp_task_wdt_reset(); 
  
  delay(900); // Total loop cycle = ~1 second
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, two **LEDs**, and a **Button** onto the canvas.
2. Wire Green LED to **GPIO12**, Red LED to **GPIO13**, and Button to **GPIO4**.
3. Paste the code and click **Run**.
4. Watch the Green LED flash once per second. The system is stable.
5. Click the button widget. The Green LED stops flashing, and the Red LED stays ON.
6. Wait 3 seconds. Watch the ESP32 automatically reset, restart, and log `TASK WATCHDOG RESET` to the Serial Monitor.

## Expected Output
Serial Monitor:
```
--- Watchdog Safety Station Booting ---
Bootup Reset Reason Code: 1 (Power-on Reset - normal boot)
Initializing TWDT...
Watchdog monitoring active. Feeding loop task.

!!! FREEZE TRIGGERED: Simulating Infinite Loop Lockup !!!
Stopping Watchdog feeds. System hanging...

(Wait 3 seconds...)

--- Watchdog Safety Station Booting ---
Bootup Reset Reason Code: 4 (!!! TASK WATCHDOG RESET - RECOVERY SUCCESSFUL !!!)
Initializing TWDT...
```

## Expected Canvas Behavior
* At startup, the Green LED flashes once per second.
* Clicking the button widget stops the Green LED and turns the Red LED ON.
* The system freezes for exactly 3 seconds, after which both LEDs turn OFF as the ESP32 resets and reboots.
* On reboot, the Red LED flashes 5 times to indicate a recovery reset before returning to normal operation.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `esp_task_wdt_init(...)` | Configures the hardware watchdog timer window and sets it to reset the CPU on timeout. |
| `esp_task_wdt_add(NULL)` | Subscribes the current running task to the watchdog timer. |
| `esp_task_wdt_reset()` | Resets (feeds) the watchdog timer, confirming the task is running normally. |
| `esp_reset_reason()` | Checks the CPU register flags on startup to identify the cause of the last reset. |

## Hardware & Safety Concept: Watchdog Timers (WDT)
Industrial controllers operating in remote environments (like satellites, weather stations, or pacemakers) cannot be manually rebooted if they freeze due to a software bug or cosmic ray bit flip. A **Watchdog Timer** is a dedicated hardware timer. During normal operation, the software must regularly reset (feed) the timer before it expires. If the software freezes or gets stuck, it stops feeding the timer. The timer then expires and triggers a hardware reset, restarting the system automatically.

## Try This! (Challenges)
1. **Interactive OLED status log**: Add an OLED display (Project 60) showing the boot status and reset reason on startup.
2. **Watchdog Alert Buzzer**: Add a buzzer on GPIO 15 that sounds a continuous alarm during lockup, before the watchdog resets the CPU.
3. **Multi-task Watchdog**: Create two tasks and subscribe both to the watchdog, ensuring that *both* tasks must run successfully to prevent a reset.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Watchdog resets the CPU during normal operation | Delay in loop is too long | Ensure the main loop runs and feeds the watchdog more often than the timeout limit (e.g. within 3 seconds) |
| WDT does not trigger reset on freeze | WDT not initialized with panic option | Verify that the second parameter of `esp_task_wdt_init()` is set to `true` |
| ESP32 boots in a loop | Watchdog setting too short | Set the timeout to at least 2 or 3 seconds to allow enough time for setup code to run |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [171 - ESP32 Multi-tasking RTOS Scheduler Demo](171-esp32-multi-tasking-rtos-scheduler-demo.md)
- [175 - ESP32 RTOS Task Notifications Alarm](175-esp32-rtos-task-notifications-alarm.md)
- [182 - ESP32 EEPROM Settings Store & Restore](182-esp32-eeprom-settings-store-and-restore.md)
