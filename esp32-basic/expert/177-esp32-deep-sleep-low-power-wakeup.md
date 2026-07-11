# 177 - ESP32 Deep Sleep Low-power Wakeup

Build an ultra-low power sensor node controller that demonstrates ESP32 Deep Sleep modes, waking up periodically using an internal RTC timer or instantly when motion is detected by a PIR sensor on an RTC GPIO pin.

## Goal
Learn how to configure deep sleep low-power states, define multiple wakeup sources (RTC timer and external GPIO interrupts), and identify boot wakeup causes.

## What You Will Build
A Blue LED is on GPIO 12, and a PIR motion sensor on GPIO 33 (an RTC-compatible GPIO). At boot, the ESP32 blinks the Blue LED 3 times and prints the wakeup reason. After 5 seconds of active operation, the ESP32 configures a 10-second timer wakeup and an external PIR wakeup, then enters Deep Sleep, shutting down the CPU and peripherals to conserve battery power.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-SR501 PIR Motion Sensor | `pir_sensor` | Yes | Yes |
| LED (Blue) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| PIR Sensor | OUT | GPIO33 | Yellow | RTC Wakeup source |
| PIR Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| Blue LED | Anode (+) | GPIO12 via 330 Ω | Blue | Wake indicator LED |
| Common ground| GND | GND | Black | Shared reference |

> **Wiring tip:** The PIR sensor must connect to GPIO 33. On the ESP32, only specific **RTC GPIO** pins (like GPIO 33) remain active during deep sleep to monitor external signals.

## Code
```cpp
// Deep Sleep Low-power Wakeup (Timer & RTC GPIO Wakeup)
#include <Arduino.h>
#include <esp_sleep.h>

const int LED_PIN = 12;
#define PIR_GPIO_PIN GPIO_NUM_33 // RTC GPIO pin

// Track boot count across deep sleep cycles using RTC memory
// This variable is stored in RTC slow memory and is preserved during sleep
RTC_DATA_ATTR int bootCount = 0;

void printWakeupReason() {
  esp_sleep_wakeup_cause_t wakeup_reason = esp_sleep_get_wakeup_cause();
  
  Serial.print("Boot count: "); Serial.println(bootCount);
  Serial.print("Wakeup cause: ");
  
  switch(wakeup_reason) {
    case ESP_SLEEP_WAKEUP_EXT0:
      Serial.println("External signal RTC_IO (PIR Motion Sensor triggered)");
      break;
    case ESP_SLEEP_WAKEUP_TIMER:
      Serial.println("RTC Timer expired (Periodic wake)");
      break;
    default:
      Serial.println("Power-on reset or software restart");
      break;
  }
}

void setup() {
  Serial.begin(115200);
  delay(500); // Let Serial Monitor open
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  bootCount++;
  
  Serial.println("\n--- Low-Power ESP32 Node Booting ---");
  printWakeupReason();
  
  // Blink LED 3 times quickly to show wake status
  for (int i = 0; i < 3; i++) {
    digitalWrite(LED_PIN, HIGH); delay(80);
    digitalWrite(LED_PIN, LOW);  delay(80);
  }
  
  // Wait 5 seconds (active simulation window)
  Serial.println("Active for 5 seconds...");
  delay(5000);
  
  // Configure Wakeup Sources:
  // 1. Timer Wakeup: Wake up every 10 seconds (10,000,000 microseconds)
  Serial.println("Enabling 10-second Timer wakeup...");
  esp_sleep_enable_timer_wakeup(10ULL * 1000000ULL);
  
  // 2. External RTC Wakeup (EXT0): Wake up if PIR sensor output (GPIO 33) goes HIGH (1)
  Serial.println("Enabling External PIR wakeup on GPIO 33...");
  esp_sleep_enable_ext0_wakeup(PIR_GPIO_PIN, 1); // 1 = HIGH, 0 = LOW
  
  // Enter Deep Sleep
  Serial.println("Entering Deep Sleep mode. Zzz...");
  delay(100); // Let Serial print finish
  
  esp_deep_sleep_start();
  
  // Execution stops here. The CPU shuts down and resets on wakeup.
}

void loop() {
  // This code is never executed because the ESP32 resets on wakeup, running setup() again.
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **PIR Sensor**, and **LED** onto the canvas.
2. Wire PIR to **GPIO33** and LED to **GPIO12**.
3. Paste the code and click **Run**.
4. Watch the Blue LED blink 3 times and count down for 5 seconds, then turn OFF (entering deep sleep).
5. Wait 10 seconds. Watch the LED flash 3 times as the timer wakes the ESP32.
6. Let it sleep again, then click the PIR sensor widget to trigger motion. Watch the ESP32 wake up instantly.

## Expected Output
Serial Monitor:
```
--- Low-Power ESP32 Node Booting ---
Boot count: 1
Wakeup cause: Power-on reset or software restart
Active for 5 seconds...
Enabling 10-second Timer wakeup...
Enabling External PIR wakeup on GPIO 33...
Entering Deep Sleep mode. Zzz...

(Wait 10 seconds...)

--- Low-Power ESP32 Node Booting ---
Boot count: 2
Wakeup cause: RTC Timer expired (Periodic wake)
```

## Expected Canvas Behavior
* At boot, the Blue LED flashes 3 times quickly.
* After 5 seconds, the system logs sleep status, and the LED turns OFF.
* After 10 seconds of inactivity, the system resets and reboots.
* Triggering the PIR widget during sleep causes the system to reboot instantly, printing `PIR Motion Sensor triggered`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `RTC_DATA_ATTR` | Directs the compiler to store the variable in RTC slow memory, preserving it during deep sleep. |
| `esp_sleep_get_wakeup_cause()` | Checks the cause of the wakeup once the CPU boots up. |
| `esp_sleep_enable_timer_wakeup(...)` | Configures the RTC timer wakeup window in microseconds. |
| `esp_sleep_enable_ext0_wakeup(...)` | Configures the RTC pin monitor to wake the CPU when the pin goes HIGH. |
| `esp_deep_sleep_start()` | Puts the CPU into deep sleep, saving power. |

## Hardware & Safety Concept: Deep Sleep Power Savings
In normal operation, the ESP32 draws 80–120 mA of current. When powered by a battery, this would deplete it in a few days. In **Deep Sleep**, the main CPU, RAM, and WiFi/Bluetooth modules are powered down. Only the ultra-low power **RTC Controller** and **RTC Memory** remain active, drawing only **10–15 µA**. When a wakeup trigger occurs (timer or pin change), the RTC controller restarts the main CPU, resetting the system.

## Try This! (Challenges)
1. **Dynamic wake duration**: Connect a potentiometer (GPIO 34) to set the sleep duration from 5 to 30 seconds before sleeping.
2. **Deep Sleep Battery monitor**: Log battery voltage levels to an SD card (Project 137) before sleeping, comparing power drain.
3. **Double PIR wakeup**: Configure two PIR sensors (GPIO 32 and 33) to wake the ESP32 on either trigger.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Boot count always resets to 1 | Variable not marked for RTC memory | Ensure `RTC_DATA_ATTR` is declared before the variable definition |
| System does not wake up from PIR | Wrong GPIO pin | Only specific RTC GPIO pins (like GPIO 32, 33, 34, 35) can be used as sleep wakeup sources |
| Serial Monitor prints garbled text on boot | Baud rate mismatch during bootup | The ESP32 ROM bootloader outputs messages at 115200 baud; ensure the terminal matches this speed |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [178 - ESP32 Light Sleep Low-power Wakeup](178-esp32-light-sleep-low-power-wakeup.md) (Next project)
- [36 - ESP32 PIR Motion Sensor Alert](../beginner/36-esp32-pir-motion-sensor-alert.md)
- [176 - ESP32 Hardware Watchdog Timer Recovery](176-esp32-hardware-watchdog-timer-recovery.md)
