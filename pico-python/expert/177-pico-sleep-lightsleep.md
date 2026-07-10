# 177 - Pico Sleep Lightsleep

Build a battery-optimised motion alarm that enters a low-power lightsleep state and wakes up immediately when a PIR motion sensor triggers a GPIO interrupt.

## Goal
Learn how to configure the Pico for low-power lightsleep using `machine.lightsleep()`, set up GPIO pins as wake-up sources, and measure wake-up cause differences in MicroPython.

## What You Will Build
A low-power security sensor node:
- **PIR Motion Sensor (GP14)**: Triggers a wake-up event on motion detection.
- **Green LED (GP15)**: Pulses when the system wakes up.
- **Red LED (GP16)**: Stays ON when the system is active; turns OFF during sleep.
- **16x2 I2C LCD (GP4, GP5)**: Displays active/sleep states and wake-up logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| PIR Motion Sensor | `button` | Yes (represented by button) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| Green & Red LEDs | `led` | Yes (two LEDs) | Yes |
| 330 Ω Resistors × 2 | `resistor` | Optional | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| PIR Sensor | Out | GP14 | Yellow | Wake-up interrupt pin |
| PIR Sensor | VCC / GND | 5V / GND | Red / Black | Power lines (requires 5V) |
| Green LED | Anode (via 330 Ω) | GP15 | Green | Wake-up trigger indicator |
| Green LED | Cathode | GND | Black | Ground return |
| Red LED | Anode (via 330 Ω) | GP16 | Red | Active state indicator |
| Red LED | Cathode | GND | Black | Ground return |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The PIR sensor output goes to GP14. The Red LED on GP16 indicates active CPU state. The LCD shares Bus 0 (GP4/GP5).

## Code
```python
from machine import Pin, I2C, lightsleep
import utime
from machine_lcd import I2cLcd

pir = Pin(14, Pin.IN, Pin.PULL_DOWN)
led_wake = Pin(15, Pin.OUT)
led_active = Pin(16, Pin.OUT)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Setup wake-up source interrupt on the PIR pin
# On RP2040, lightsleep can be woken by a pin change if an IRQ is configured.
def wakeup_handler(pin):
    pass  # We just need the interrupt to wake up, handling is in the main loop

pir.irq(trigger=Pin.IRQ_RISING, handler=wakeup_handler)

lcd.clear()
lcd.putstr("Lightsleep Demo\nInitialising...")
utime.sleep(1.0)

wake_count = 0

print("Low-power lightsleep node active.")

while True:
    # 1. Active State
    led_active.value(1)
    led_wake.value(0)
    
    lcd.backlight_on()
    lcd.clear()
    lcd.putstr("System Awake\nStatus: Active  ")
    utime.sleep(3.0)  # Stay active for 3 seconds to process task
    
    # 2. Prepare for sleep
    lcd.clear()
    lcd.putstr("Entering Sleep...\nPower Saving...")
    utime.sleep(1.0)
    
    lcd.backlight_off()  # Turn off LCD backlight to save power
    led_active.value(0)  # Turn off status LED
    
    print(">> Sleep active. Waiting for PIR trigger or 10s timeout...")
    
    # 3. Enter Lightsleep
    # Pass sleep duration in milliseconds (10 seconds)
    # The Pico halts execution here until 10s expires OR PIR goes HIGH
    lightsleep(10000)
    
    # 4. Wake-Up Recovery
    wake_count += 1
    
    # Determine wake-up cause by reading the PIR pin immediately
    if pir.value() == 1:
        cause_msg = "Woken by: PIR"
        led_wake.value(1)  # Flash green LED for motion
        print(">> WAKEUP: Motion detected!")
    else:
        cause_msg = "Woken by: TIMEOUT"
        print(">> WAKEUP: 10-second timer elapsed.")
        
    lcd.backlight_on()
    lcd.clear()
    lcd.putstr("WAKEUP #{}\n".format(wake_count))
    lcd.putstr(cause_msg)
    
    utime.sleep(2.0)  # Hold wake-up message
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **PIR Sensor** (represented by button), **two LEDs**, and **I2C LCD** onto the canvas.
2. Connect PIR to **GP14**, LEDs to **GP15/GP16**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the Pico enter sleep (OLED/LCD backlights turn off).
5. Click the PIR button within 10 seconds. Verify that the system wakes up instantly, flashes the Green LED, and shows `Woken by: PIR`.

## Expected Output
Terminal:
```
Low-power lightsleep node active.
>> Sleep active. Waiting for PIR trigger or 10s timeout...
>> WAKEUP: Motion detected!
>> Sleep active. Waiting for PIR trigger or 10s timeout...
>> WAKEUP: 10-second timer elapsed.
```

## Expected Canvas Behavior
* System turns ON, LCD reads `System Awake`. Red LED (GP16) turns ON.
* After 3 seconds, LCD backlight turns OFF, Red LED turns OFF.
* Pressing the PIR button on GP14 instantly turns the LCD backlight back ON, turns the Green LED (GP15) ON, and displays `Woken by: PIR`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `lightsleep(10000)` | Suspends the processor clocks and halts execution, waking up after 10000 ms or upon a pin interrupt. |
| `pir.irq(...)` | Configures the interrupt trigger on GP14, which is necessary to wake the processor from lightsleep. |

## Hardware & Safety Concept: Lightsleep vs Deepsleep
In **lightsleep**, the processor's clock is gated, but RAM contents are preserved, allowing code to resume instantly from the exact line where sleep was called. In **deepsleep**, the processor is powered down entirely, RAM is lost, and the chip restarts from the top of the program upon wake-up. Lightsleep is best for fast wake-up times and maintaining variables, whereas deepsleep achieves the lowest possible current draw.

## Try This! (Challenges)
1. **Critical Buzzer Chirp**: Sound a buzzer on GP9 for 500 ms when woken by the PIR sensor to deter intruders.
2. **Snooze Timer**: Add a button on GP13. When pressed, increase the lightsleep timer from 10 seconds to 30 seconds (sleep extension).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| System wakes up immediately after entering sleep | Pin floating | Ensure the PIR pin has a solid pull-down resistor (`Pin.PULL_DOWN` or external 10k to GND) so noise doesn't trigger fake wakes. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [36 - Pico PIR Motion Sensor alert](../../beginner/36-pico-pir-motion-sensor-alert.md)
- [149 - Pico PIR Timed Security Light Relay LCD](../advanced/149-pico-pir-timed-security-light-relay-lcd.md)
