# 178 - Pico Sleep Deepsleep

Build an ultra-low-power environmental datalogger that stores boot counts in flash memory, enters deepsleep, and restarts when an external button triggers a reset wake-up.

## Goal
Learn how to implement deepsleep states using `machine.deepsleep()`, persist data across hardware resets by writing to the flash filesystem, and analyze the boot wake-up cause in MicroPython.

## What You Will Build
An ultra-low-power datalogger:
- **Button (GP14)**: Triggers wake-up from deepsleep immediately.
- **Red LED (GP16)**: Turns ON during active telemetry scanning; turns OFF in deepsleep.
- **SSD1306 OLED (GP4, GP5)**: Displays the boot log cause and current cycle count.
- **Flash Storage (`bootlog.txt`)**: Saves the cumulative count of deepsleep cycles.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| SSD1306 OLED Display | `oled` | Yes | Yes |
| Red LED | `led` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Button | Terminal 1 | GP14 | White | Wake-up trigger pin |
| Button | Terminal 2 | GND | Black | Ground return |
| Red LED | Anode (via 330 Ω) | GP16 | Red | Active CPU indicator |
| Red LED | Cathode | GND | Black | Ground return |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The button on GP14 acts as the wake-up trigger. The OLED screen uses GP4/GP5. In deepsleep, almost all Pico peripherals are powered down.

## Code
```python
from machine import Pin, I2C, deepsleep, reset_cause
import utime, machine, ssd1306

led_active = Pin(16, Pin.OUT)
btn_wake = Pin(14, Pin.IN, Pin.PULL_UP)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# Configure the wake-up interrupt on the button pin
# To wake the Pico from deepsleep, the pin must have an active interrupt.
def wakeup_dummy(pin):
    pass
btn_wake.irq(trigger=Pin.IRQ_FALLING, handler=wakeup_dummy)

# 1. Read persistent wake count from flash filesystem
def get_wake_count():
    try:
        with open('bootlog.txt', 'r') as f:
            return int(f.read().strip())
    except Exception:
        return 0

def save_wake_count(count):
    try:
        with open('bootlog.txt', 'w') as f:
            f.write(str(count))
    except Exception as e:
        print("Flash write error:", e)

# 2. Determine boot reason
cause = reset_cause()
# machine.DEEPSLEEP_RESET = 4 (on RP2040)
if cause == machine.DEEPSLEEP_RESET:
    boot_cause = "DEEPSLEEP WAKEUP"
    wake_count = get_wake_count() + 1
else:
    boot_cause = "POWER-ON RESET"
    wake_count = 0  # Reset count on fresh power-up

save_wake_count(wake_count)

# 3. Active operation: turn ON active LED and update OLED
led_active.value(1)
oled.fill(0)
oled.rect(0, 0, 128, 14, 1)
oled.text("DEEPSLEEP NODE", 10, 3, 0)

oled.text("Boot  : {}".format(boot_cause[:15]), 10, 22, 1)
oled.text("Cycle : {}".format(wake_count), 10, 36, 1)
oled.text("Sleeping in 4s..", 10, 50, 1)
oled.rect(0, 0, 128, 64, 1)
oled.show()

print("Boot cause:", boot_cause)
print("Deepsleep cycle count:", wake_count)

# Keep the system awake for 4 seconds to allow flash reads/writes and display
utime.sleep(4.0)

# 4. Enter deep sleep
oled.fill(0)
oled.text("DEEPSLEEP ACTIVE", 2, 28, 1)
oled.show()
utime.sleep(0.5)

oled.fill(0)
oled.show()  # Turn off display content
led_active.value(0)  # Turn off active LED

print(">> Entering deepsleep now. Wake up with button or after 8s...")

# Enter deep sleep for 8 seconds (8000 ms)
# Upon wake-up, the Pico will fully reboot and start execution from line 1!
deepsleep(8000)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **SSD1306 OLED**, **Push Button**, and **Red LED** onto the canvas.
2. Connect Button to **GP14**, LED to **GP16**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. The screen displays `Cause: POWER-ON RESET`.
5. After 4 seconds, the system enters deepsleep (LED and OLED turn off). Press the Button on **GP14** within 8 seconds to wake the Pico. The system reboots, and the cycle count increments to `1`.

## Expected Output
Terminal:
```
Boot cause: POWER-ON RESET
Deepsleep cycle count: 0
>> Entering deepsleep now. Wake up with button or after 8s...
(system resets)
Boot cause: DEEPSLEEP WAKEUP
Deepsleep cycle count: 1
>> Entering deepsleep now. Wake up with button or after 8s...
```

## Expected Canvas Behavior
* Boot: Red LED turns ON. OLED displays `DEEPSLEEP NODE`.
* 4 seconds: OLED and Red LED turn OFF. System enters deepsleep.
* Pressing GP14 button: Red LED turns ON, system reboots, and the cycle count increases.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `deepsleep(8000)` | Shuts down CPU clocks, PLLs, and SRAM power, waking up on pin transition or after 8 seconds. |
| `reset_cause()` | Checks if the system boot was triggered by a power cycle or a deepsleep wake-up event. |

## Hardware & Safety Concept: Deepsleep Persistence
In deepsleep, the processor core is powered off and all RAM is lost. To persist state variables (like log records or charging metrics) across deepsleep cycles, systems must write these variables to non-volatile flash memory before entering sleep. On reboot, the program reads the flash file to recover its operating state, behaving as a continuous run.

## Try This! (Challenges)
1. **Datalogger Simulation**: Connect an LDR on GP26 and write the measured light value to a file (`lightlog.csv`) on every deepsleep wake-up cycle.
2. **Alert Strobe**: Flash a buzzer on GP15 three times if the system reboots due to an unexpected power loss instead of a scheduled deepsleep wake-up.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Cycle count stays at 0 | Power-on cause registered | The Pico reboots on deepsleep wake-up. If `reset_cause()` does not return `WDT_RESET` or `DEEPSLEEP_RESET` on your board, print the raw integer of `reset_cause()` to adjust the check. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [176 - Pico Watchdog Timer Recovery](176-pico-watchdog-timer-recovery.md)
- [177 - Pico Sleep Lightsleep](177-pico-sleep-lightsleep.md)
