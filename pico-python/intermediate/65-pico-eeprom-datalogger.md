# 65 - Pico EEPROM Datalogger

Store variables across power cycles in simulated flash memory (EEPROM) using the MicroPython filesystem.

## Goal
Learn how to read and write data to the Pico's non-volatile internal flash memory using the file system APIs in MicroPython.

## What You Will Build
A non-volatile parameter store:
- **Serial Interface**: Increments a boot counter and stores it in a configuration file (`config.txt`). The value is retrieved on boot and updated, persisting even if the Pico is reset.
- **Button (GP14)**: Clears the boot counter when pressed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes (reset switch) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Terminal 1 | GP14 | Blue | Reset button input (reads LOW on press) |
| Push Button | Terminal 2 | GND | Black | Shorts GP14 to GND on press |

> **Wiring tip:** The button connects between GP14 and GND, using the internal pull-up on GP14.

## Code
```python
from machine import Pin
import utime
import os

btn_reset = Pin(14, Pin.IN, Pin.PULL_UP)

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

print("Active loop started. Press the reset button on GP14 to clear the counter.")

while True:
    # Check for reset trigger
    if btn_reset.value() == 0:
        boot_count = 0
        with open(CONFIG_FILE, "w") as f:
            f.write("0")
        print(">> Config file cleared and reset to 0.")
        utime.sleep(1) # Lockout delay
        
    utime.sleep_ms(100) # Polling update speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Push Button** onto the canvas.
2. Connect Button Terminal 1 to **GP14** and Terminal 2 to **GND**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the button component on the canvas to reset the boot counter.
5. Click **Stop** and **Run** again to see the boot count persist and increment.

## Expected Output
First Boot:
```
Config file not found. Initializing...
Boot count loaded from flash memory: 1
Active loop started. Press the reset button on GP14 to clear the counter.
```
Second Boot (after Stop & Run):
```
Boot count loaded from flash memory: 2
Active loop started. Press the reset button on GP14 to clear the counter.
```

## Expected Canvas Behavior
- The serial terminal prints the updated boot counter and registers the reset action when the button is clicked.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `open(CONFIG_FILE, "r")` | Opens the configuration file in read mode to retrieve the stored counter. |
| `open(CONFIG_FILE, "w")` | Opens the configuration file in write mode to save the updated counter to flash. |
| `f.write(str(boot_count))` | Writes the string representation of the counter to the file. |

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
This project runs in MbedO **MicroPython interpreted mode**. The virtual file system is simulated in memory.

## Related Projects
- [03 - Pico Button LED](03-pico-button-led.md)
- [13 - Pico Button Counter](13-pico-button-counter.md)
