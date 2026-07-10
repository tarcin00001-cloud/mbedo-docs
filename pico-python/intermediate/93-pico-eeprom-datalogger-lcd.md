# 93 - Pico EEPROM Datalogger LCD

Build a boot counter log console that saves execution counts to internal flash and displays the running statistics on an I2C LCD screen.

## Goal
Learn how to read and write parameters to the Pico's non-volatile internal flash file system, and output formatting variables to an I2C LCD in MicroPython.

## What You Will Build
A flash parameter monitor:
- **Serial Interface**: Increments a boot counter and stores it in a configuration file (`config.txt`). The value is retrieved on boot and updated, persisting even if the Pico is reset.
- **Button (GP14)**: Clears the boot counter when pressed.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the boot count and the running uptime.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes (reset switch) |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Terminal 1 | GP14 | Blue | Reset button input (reads LOW on press) |
| Push Button | Terminal 2 | GND | Black | Shorts GP14 to GND on press |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the button between GP14 and GND. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
import os
from machine_lcd import I2cLcd

btn_reset = Pin(14, Pin.IN, Pin.PULL_UP)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

CONFIG_FILE = "config.txt"

# 1. Read boot counter from flash file
boot_count = 0

try:
    with open(CONFIG_FILE, "r") as f:
        data = f.read().strip()
        if data:
            boot_count = int(data)
except OSError:
    # File does not exist on first boot, create it
    print("Config file not found. Initializing...")
    with open(CONFIG_FILE, "w") as f:
        f.write("0")

# 2. Increment boot counter
boot_count += 1
print("Boot count loaded from flash memory:", boot_count)

# Save new value to flash file
with open(CONFIG_FILE, "w") as f:
    f.write(str(boot_count))

# Update LCD Display
lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Boot Count: " + str(boot_count))
lcd.move_to(0, 1)
lcd.putstr("Uptime: 0s")

start_time = utime.ticks_ms()

while True:
    # Calculate elapsed runtime in seconds
    elapsed = utime.ticks_diff(utime.ticks_ms(), start_time) // 1000
    
    # Update LCD second row
    lcd.move_to(0, 1)
    lcd.putstr("Uptime: " + str(elapsed) + "s   ")
    
    # Check for reset trigger
    if btn_reset.value() == 0:
        boot_count = 0
        with open(CONFIG_FILE, "w") as f:
            f.write("0")
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Memory Cleared")
        lcd.move_to(0, 1)
        lcd.putstr("Boot Count: 0")
        print(">> Config file cleared and reset to 0.")
        utime.sleep(1) # Lockout delay
        
    utime.sleep_ms(250) # Polling update speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button**, and **I2C LCD** onto the canvas.
2. Connect Button to **GP14** and LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the LCD display.
5. Click **Stop** and **Run** again to see the boot count persist and increment on the LCD.

## Expected Output
```
Boot count loaded from flash memory: 1
```
(On screen: "Boot Count: X" and "Uptime: Ys" on respective lines, updating.)

## Expected Canvas Behavior
- The LCD component displays the incremented boot counter on startup, and updates the uptime tracker in real time.
- Clicking the button resets the boot count to 0.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `open(CONFIG_FILE, "r")` | Opens the configuration file in read mode to retrieve the stored counter. |
| `open(CONFIG_FILE, "w")` | Opens the configuration file in write mode to save the updated counter to flash. |
| `lcd.putstr("Boot Count: " + ...)` | Prints the loaded count value on the LCD display. |

## Hardware & Safety Concept: Flash Wear and File Systems
Microcontrollers use flash memory to store program code and user data. Standard flash memory has a write endurance limit (typically **10,000 to 100,000 write cycles** per block). Writing to files repeatedly in a fast loop can cause the flash memory to wear out quickly. To prevent this, data should only be written to flash when changes occur, rather than on every loop iteration.

## Try This! (Challenges)
1. **Latching LED Indicator**: Connect an LED on GP15 and toggle its state ON/OFF based on the last state stored in the configuration file on boot.
2. **Datalogger**: Append sensor readings (like temperature or light level) to a CSV log file (`datalog.csv`) every 5 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Error on boot: `ValueError` | File contains empty or invalid data | Manually delete the config file using `os.remove("config.txt")` in the terminal to clear invalid data. |
| Values do not update after reboot | File not closed properly | Always use the `with` block context manager when opening files, which closes the file automatically. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [65 - Pico EEPROM Datalogger](65-pico-eeprom-datalogger.md)
- [92 - Pico Relay Button LCD](92-pico-relay-button-lcd.md)
