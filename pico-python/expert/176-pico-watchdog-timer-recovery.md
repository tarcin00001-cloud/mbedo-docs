# 176 - Pico Watchdog Timer Recovery

Build a self-recovering system that uses a hardware Watchdog Timer (WDT) to detect software freezes and automatically reset the Pico, logging the cause of the boot on an OLED.

## Goal
Learn how to initialize and feed a Watchdog Timer in MicroPython, simulate system crashes, and identify the boot cause (power-on vs. watchdog reset) using `machine.reset_cause()`.

## What You Will Build
A self-healing system:
- **Watchdog Timer (WDT)**: Monitors the loop and resets the Pico if it freezes for more than 2 seconds.
- **Green LED (GP15)**: Blinks on every successful watchdog feed.
- **Red LED (GP16)**: Illuminates if the system is forced to freeze.
- **Button A (GP13)**: Simulates a system crash (infinite loop).
- **SSD1306 OLED (GP4, GP5)**: Displays system health status and identifies the boot cause.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| SSD1306 OLED Display | `oled` | Yes | Yes |
| Green & Red LEDs | `led` | Yes (two LEDs) | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| 330 Ω Resistors × 2 | `resistor` | Optional | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Green LED | Anode (via 330 Ω) | GP15 | Green | Watchdog feed pulse |
| Green LED | Cathode | GND | Black | Ground return |
| Red LED | Anode (via 330 Ω) | GP16 | Red | Freeze indicator |
| Red LED | Cathode | GND | Black | Ground return |
| Button A | Terminal 1 | GP13 | White | Simulates system crash |
| Button A | Terminal 2 | GND | Black | Ground return |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The OLED screen shares GP4/GP5. The crash simulator button on GP13 uses an internal pull-up. The Green and Red LEDs connect to GP15 and GP16.

## Code
```python
from machine import Pin, I2C, WDT, reset_cause
import utime, machine, ssd1306

led_feed = Pin(15, Pin.OUT)
led_freeze = Pin(16, Pin.OUT)
btn_crash = Pin(13, Pin.IN, Pin.PULL_UP)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# Determine the cause of the reset
# machine.PWRON_RESET = 1, machine.WDT_RESET = 3 (on RP2040)
cause = reset_cause()
boot_msg = "POWER-ON RESET"
if cause == machine.WDT_RESET:
    boot_msg = "WATCHDOG RECOVERY"

oled.fill(0)
oled.rect(0, 0, 128, 14, 1)
oled.text("BOOT DIAGNOSTICS", 2, 3, 0)
oled.text("Cause:", 10, 24, 1)
oled.text(boot_msg, 10, 38, 1)
oled.show()
utime.sleep(2.0)  # Hold diagnostic screen

# Start Watchdog Timer with a 2-second timeout
# WDT is fed in the main loop. If not fed within 2s, the Pico resets.
wdt = WDT(timeout=2000)
print("Watchdog Timer active (2000ms timeout).")

while True:
    now = utime.ticks_ms()
    
    # Check if user wants to simulate a system crash (Button A)
    if btn_crash.value() == 0:
        print(">> CRASH SIMULATION TRIGGERED! Freezing system...")
        led_freeze.value(1)
        led_feed.value(0)
        
        # Draw freeze screen
        oled.fill(0)
        oled.rect(0, 0, 128, 64, 1)
        oled.text("SYSTEM FROZEN", 12, 20, 1)
        oled.text("No WDT feeding!", 4, 36, 1)
        oled.show()
        
        # Enter an infinite blocking loop. WDT will trigger a reset in 2 seconds.
        while True:
            pass

    # --- Normal operation: Feed the Watchdog ---
    wdt.feed()
    
    # Toggle feed indicator
    led_feed.value(1)
    
    # Draw normal screen
    oled.fill(0)
    oled.rect(0, 0, 128, 14, 1)
    oled.text("SYSTEM RUNNING", 10, 3, 0)
    
    # Display countdown estimate
    oled.text("WDT status: FED", 10, 24, 1)
    oled.text("Press BTN to", 10, 38, 1)
    oled.text("simulate crash", 10, 50, 1)
    oled.rect(0, 0, 128, 64, 1)
    oled.show()
    
    utime.sleep_ms(100)
    led_feed.value(0)
    utime.sleep_ms(400)  # Loop repeats every 500 ms (well within 2000 ms limit)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **SSD1306 OLED**, **Push Button**, and **two LEDs** onto the canvas.
2. Connect Button to **GP13**, LEDs to **GP15/GP16**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the system boot and display `POWER-ON RESET`.
5. Press the Button on **GP13** to freeze the system. Watch the red LED turn ON, and wait for 2 seconds. The Pico will reset and reboot, displaying `WATCHDOG RECOVERY`.

## Expected Output
Terminal:
```
Watchdog Timer active (2000ms timeout).
>> CRASH SIMULATION TRIGGERED! Freezing system...
(microcontroller resets)
Watchdog Timer active (2000ms timeout).
```

## Expected Canvas Behavior
* Startup: OLED reads `BOOT DIAGNOSTICS` / `Cause: POWER-ON RESET` for 2 seconds, then switches to `SYSTEM RUNNING`.
* Green LED (GP15) pulses regularly to indicate watchdog feeding.
* Press Button (GP13): Red LED (GP16) turns ON, Green LED stops pulsing, OLED displays `SYSTEM FROZEN`.
* After 2 seconds, the simulator resets, and the OLED displays `Cause: WATCHDOG RECOVERY`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `WDT(timeout=2000)` | Initializes the hardware watchdog timer peripheral on the RP2040. |
| `wdt.feed()` | Resets the watchdog internal countdown timer back to the maximum timeout value. |
| `reset_cause()` | Reads the boot cause register to see what triggered the last reset. |

## Hardware & Safety Concept: Watchdog Timers in High-Reliability Systems
Watchdog timers are critical safety devices in high-reliability applications (like space exploration, medical systems, or automotive ECUs). If a stray cosmic ray flips a memory bit or a software bug causes an infinite loop, the device could hang forever. WDTs ensure that the system automatically resets and recovers to a safe operating state without human intervention.

## Try This! (Challenges)
1. **Error Log Storage**: Write the reboot count to flash memory (`reboots.txt`) each time the system starts up after a WDT reset, and display the count on screen.
2. **Pre-warn indicator**: Sound a warning buzzer on GP14 if the loop time exceeds 1.5 seconds, alerting that a crash is imminent.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The system continuously loops and resets | WDT timeout is too short | The loop sleep time (`utime.sleep_ms`) must be significantly shorter than the WDT timeout. Ensure `wdt.feed()` is called frequently. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [119 - OLED Digital Stop Watch (Start/Stop buttons)](../intermediate/119-pico-oled-digital-stop-watch-start-stop-buttons.md)
- [141 - Pico I2C EEPROM Data Logger DHT22 LCD](../advanced/141-pico-i2c-eeprom-data-logger-dht22-lcd.md)
