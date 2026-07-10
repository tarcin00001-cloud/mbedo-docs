# 184 - Pico MicroPython Flash File Logger

Build an on-board datalogger that writes climate records directly to the Pico's internal flash memory filesystem in CSV format, displays logs, and handles flash limits.

## Goal
Learn how to use MicroPython's file I/O operations to create, append, read, and delete files on the internal flash filesystem, manage storage safety, and monitor file size using `os.stat()`.

## What You Will Build
An internal flash memory logger station:
- **DHT22 Climate Sensor (GP12)**: Measures temperature and humidity.
- **Save Button (GP13)**: Triggers an immediate climate write to the internal flash file.
- **Erase Button (GP14)**: Deletes the log file, clearing space on the filesystem.
- **Write Indicator LED (GP15)**: Flashes when data is being written to flash.
- **SSD1306 OLED (GP4, GP5)**: Displays current climate, CSV file size in bytes, and total records.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| SSD1306 OLED Display | `oled` | Yes | Yes |
| Tactile Push Buttons × 2 | `button` | Yes | Yes |
| Green LED | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | DATA | GP12 | Yellow | Climate sensor data |
| DHT22 | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| Save Button | Terminal 1 | GP13 | White | Data log trigger |
| Save Button | Terminal 2 | GND | Black | Ground return |
| Erase Button | Terminal 1 | GP14 | Yellow | Delete file trigger |
| Erase Button | Terminal 2 | GND | Black | Ground return |
| Green LED | Anode (via 330 Ω) | GP15 | Green | Write operation flash |
| Green LED | Cathode | GND | Black | Ground return |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the Save Button to GP13 and the Erase Button to GP14, both with internal pull-ups. The write flash LED is on GP15.

## Code
```python
from machine import Pin, I2C
import utime, dht, os, ssd1306

dht_sensor = dht.DHT22(Pin(12))
btn_save = Pin(13, Pin.IN, Pin.PULL_UP)
btn_erase = Pin(14, Pin.IN, Pin.PULL_UP)
led_write = Pin(15, Pin.OUT)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

LOG_FILE = "flashlog.csv"
MAX_FILE_SIZE = 4000  # Prevent flash exhaustion (4 KB limit for demo)

def get_file_size():
    """Returns the size of the log file in bytes, or 0 if it doesn't exist."""
    try:
        # os.stat returns a tuple. Index 6 is file size in bytes.
        return os.stat(LOG_FILE)[6]
    except OSError:
        return 0

def write_header():
    """Writes the CSV header if the file does not exist."""
    size = get_file_size()
    if size == 0:
        try:
            with open(LOG_FILE, "w") as f:
                f.write("Time_ms,Temp_C,Hum_pct\n")
            print("CSV Header initialized in flash.")
        except OSError as e:
            print("Header write failed:", e)

def log_data(temp, humid):
    """Appends a new climate log line to the flash file."""
    # Check flash safety limit
    current_size = get_file_size()
    if current_size >= MAX_FILE_SIZE:
        print("Warning: Log file size limit reached! Erase logs to continue.")
        return False
        
    led_write.value(1)
    timestamp = utime.ticks_ms()
    try:
        with open(LOG_FILE, "a") as f:
            f.write("{},{:.1f},{:.1f}\n".format(timestamp, temp, humid))
        print("Logged: {} ms, {:.1f} C, {:.1f} %".format(timestamp, temp, humid))
        utime.sleep_ms(100)  # Visual flash pulse
        led_write.value(0)
        return True
    except OSError as e:
        print("Flash write error:", e)
        led_write.value(0)
        return False

def delete_logs():
    """Deletes the log file from flash."""
    try:
        os.remove(LOG_FILE)
        print("Log file deleted from flash.")
    except OSError:
        print("No log file found to delete.")

# Initialise file
write_header()

oled.fill(0)
oled.text("Flash Logger", 16, 20)
oled.text("Online", 16, 36)
oled.show()
utime.sleep(1.0)

print("Flash file logger ready.")

last_btn = 0
records_count = 0

while True:
    now = utime.ticks_ms()
    
    # Read DHT22
    try:
        dht_sensor.measure()
        t = dht_sensor.temperature()
        h = dht_sensor.humidity()
        sensor_ok = True
    except OSError:
        t, h = 0.0, 0.0
        sensor_ok = False

    # Manual Save Button (GP13)
    if btn_save.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        last_btn = now
        if sensor_ok:
            if log_data(t, h):
                records_count += 1
        else:
            print("Save failed: Sensor error!")
            
    # Manual Erase Button (GP14)
    if btn_erase.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        last_btn = now
        delete_logs()
        write_header()
        records_count = 0
        
    # Get current file size
    f_size = get_file_size()
    
    # Update display
    oled.fill(0)
    oled.rect(0, 0, 128, 14, 1)
    oled.text("FLASH DATA LOGGER", 4, 3, 0)
    
    if sensor_ok:
        oled.text("T:{:.1f}C H:{:.0f}%".format(t, h), 10, 20, 1)
    else:
        oled.text("SENSOR ERROR", 10, 20, 1)
        
    oled.text("File: {} B".format(f_size), 10, 34, 1)
    oled.text("Logs: {}/{}".format(records_count, MAX_FILE_SIZE), 10, 48, 1)
    
    oled.rect(0, 0, 128, 64, 1)
    oled.show()
    
    utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **SSD1306 OLED**, **two Buttons**, and **LED** onto the canvas.
2. Connect DHT22 to **GP12**, Save Button to **GP13**, Erase Button to **GP14**, LED to **GP15**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Press the Save Button on **GP13**. Verify that the LED flashes and `File: 56 B` appears, indicating file growth.
5. Press the Erase Button on **GP14** and check if the file size resets.

## Expected Output
Terminal:
```
CSV Header initialized in flash.
Flash file logger ready.
Logged: 5320 ms, 24.5 C, 50.2 %
Logged: 7840 ms, 24.6 C, 50.1 %
Log file deleted from flash.
CSV Header initialized in flash.
```

## Expected Canvas Behavior
* Boot: OLED shows `File: 24 B` (the header size).
* Save Button click: LED (GP15) flashes, `File` size increases, and `Logs` count increments.
* Erase Button click: `File` size resets to 24 B, `Logs` counter goes to 0.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `os.stat(LOG_FILE)[6]` | Calls the filesystem status query and extracts the file size in bytes from the tuple. |
| `open(LOG_FILE, "a")` | Opens the target file in append mode, writing new data to the end of the file. |

## Hardware & Safety Concept: Flash Wear and File Corruption
The Pico's RP2040 uses external SPI flash memory (typically 2 MB). Flash memory has a limited number of write cycles (typically 10,000 to 100,000 erase cycles per block). Writing data continuously inside a fast loop will quickly destroy the flash cells. Dataloggers must use **throttled writes** (e.g. saving once every 5 minutes or holding data in RAM before writing blocks) to protect the hardware from premature wear.

## Try This! (Challenges)
1. **Critical Storage Alarm**: Sound a warning buzzer on GP16 if the file size exceeds 3500 bytes (storage almost full).
2. **Auto-Logger**: Log temperature and humidity automatically every 5 seconds without clicking the button, but stop immediately if the file size exceeds 3500 bytes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| File size reads 0 after writing | File was not closed | MicroPython buffers writes. If the file is not closed (`f.close()`), data is lost on reset. Using the `with` statement ensures the file is closed automatically. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [137 - Pico SD Card Temperature Logger (DHT22 → CSV file)](137-pico-i2c-eeprom-data-logger-dht22-lcd.md)
- [155 - Pico Environmental Data Station SD Card Logger](../advanced/155-pico-environmental-data-station-sd-card-logger.md)
