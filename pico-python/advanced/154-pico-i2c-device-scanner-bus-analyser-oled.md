# 154 - Pico I2C Device Scanner Bus Analyser OLED

Build an I2C bus analyser that scans the I2C bus for all connected devices, displays their hexadecimal addresses on an SSD1306 OLED in a grid layout, identifies known devices by address, and monitors for bus changes on repeat scans.

## Goal
Learn how to perform an I2C bus scan in MicroPython, map detected addresses to a device identification table, render a scrollable address grid on an SSD1306 OLED, and detect when devices are added or removed between scans.

## What You Will Build
An I2C bus analyser:
- **SSD1306 OLED (GP4, GP5)**: Renders the address grid and device IDs.
- **Multiple I2C Devices (GP4, GP5 shared bus)**: Any combination — OLED, LCD, BMP180, MPU-6050, AT24C32, etc.
- **Button (GP13)**: Triggers an immediate rescan.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |
| Various I2C Devices | Any I2C modules | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 — always present |
| BMP180 (optional) | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Appears as 0x77 |
| MPU-6050 (optional) | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Appears as 0x68 |
| AT24C32 (optional) | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Appears as 0x50 |
| I2C LCD (optional) | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Appears as 0x27 or 0x3F |
| Button | Terminal 1 / 2 | GP13 / GND | White / Black | Force rescan |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** All I2C devices share GP4 (SDA) and GP5 (SCL). Each device must have a unique 7-bit address. The OLED itself (0x3C) will always appear in the scan results. Button on GP13 with PULL_UP.

## Code
```python
from machine import Pin, I2C
import utime, ssd1306

i2c  = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)
btn  = Pin(13, Pin.IN, Pin.PULL_UP)

# Known I2C device address map
KNOWN_DEVICES = {
    0x3C: "OLED    ",
    0x3D: "OLED-2  ",
    0x27: "LCD     ",
    0x3F: "LCD-2   ",
    0x68: "MPU6050 ",
    0x69: "MPU6050B",
    0x76: "BMP280  ",
    0x77: "BMP180  ",
    0x50: "EEPROM  ",
    0x51: "EEPROM-2",
    0x48: "ADS1115 ",
    0x5A: "MLX90614",
    0x40: "HTU21D  ",
    0x70: "TCA9548 ",
}

SCAN_INTERVAL_MS = 5000
last_scan     = utime.ticks_ms() - SCAN_INTERVAL_MS
prev_devices  = set()
found_devices = []
new_found     = set()
lost_devices  = set()
page          = 0
last_btn      = 0

def do_scan():
    global found_devices, new_found, lost_devices, prev_devices, page
    raw          = i2c.scan()
    current_set  = set(raw)
    new_found    = current_set - prev_devices
    lost_devices = prev_devices - current_set
    prev_devices = current_set
    found_devices = sorted(raw)
    page = 0
    print("Scan found {} devices: {}".format(
        len(found_devices), [hex(a) for a in found_devices]))

def draw_results():
    oled.fill(0)
    oled.text("I2C SCANNER", 16, 0)
    oled.hline(0, 10, 128, 1)

    if not found_devices:
        oled.text("No devices!", 16, 28)
        oled.show(); return

    # Show up to 4 devices per page (2 columns × 2 rows)
    items_per_page = 4
    start = page * items_per_page
    shown = found_devices[start:start + items_per_page]

    for i, addr in enumerate(shown):
        col = i % 2
        row = i // 2
        x   = col * 64
        y   = 14 + row * 24
        name = KNOWN_DEVICES.get(addr, "Unknown ")
        is_new = addr in new_found
        flag = "+" if is_new else " "
        oled.text("{}0x{:02X}".format(flag, addr), x, y)
        oled.text(name[:8], x, y + 10)

    total_pages = max(1, (len(found_devices) + items_per_page - 1) // items_per_page)
    oled.text("Pg{}/{} N:{}".format(page + 1, total_pages, len(found_devices)), 0, 56)
    oled.show()

# Initial scan
do_scan()
draw_results()
print("I2C bus analyser active. Press button to rescan.")

while True:
    now = utime.ticks_ms()

    # Auto rescan
    if utime.ticks_diff(now, last_scan) >= SCAN_INTERVAL_MS:
        last_scan = now
        do_scan()
        draw_results()
        if new_found:
            print("New devices:", [hex(a) for a in new_found])
        if lost_devices:
            print("Lost devices:", [hex(a) for a in lost_devices])

    # Button: manual rescan
    if btn.value() == 0 and utime.ticks_diff(now, last_btn) > 500:
        last_btn = now
        do_scan()
        draw_results()

    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag **Pico**, **SSD1306 OLED**, and optionally **BMP180**, **MPU-6050**, **AT24C32**, or **I2C LCD** onto the canvas.
2. Connect all I2C devices to **GP4/GP5** (shared bus). Connect Button to **GP13**. Connect power and GND.
3. Paste code, click **Run**. The OLED shows detected addresses and device names.
4. Add/remove devices (in MbedO: enable/disable components). Press Button to rescan — new devices appear with a `+` prefix.

## Expected Output
```
Scan found 3 devices: ['0x3c', '0x68', '0x77']
I2C bus analyser active. Press button to rescan.
New devices: ['0x50']
```
(On OLED: 4-up address grid showing `+0x50 EEPROM` when the EEPROM is newly connected.)

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `i2c.scan()` | Attempts communication with every address 0x00–0x7F and returns a list of addresses that acknowledged. |
| `new_found = current_set - prev_devices` | Set difference detects addresses present in the new scan but not the previous one — newly connected devices. |

## Hardware & Safety Concept: I2C Address Conflicts
I2C devices have fixed or semi-configurable addresses. When two devices share the same address, both respond to reads/writes simultaneously, causing data corruption. Always check device datasheets for address ranges. Use an I2C multiplexer (TCA9548A, address 0x70) to connect multiple identical devices that cannot be re-addressed.

## Try This! (Challenges)
1. **Register Read**: For detected known devices, read a known register (e.g. MPU-6050 WHO_AM_I at 0x75) and display the response value as a verification.
2. **Bus Speed Test**: After scanning, measure the maximum clock frequency by incrementing `freq` until `i2c.scan()` returns an empty list.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED shows "No devices" | Wrong I2C bus | Verify GP4/GP5 are connected and `I2C(0,...)` matches the hardware connections. |
| Address appears twice | Duplicate device | Check if two modules have the same address — refer to their address configuration pins. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [136 - Pico DS18B20 Multi-Probe Temperature Network LCD](136-pico-ds18b20-multi-probe-temperature-network-lcd.md)
- [140 - Pico Full Weather Console DHT22 BMP180 LDR OLED](140-pico-full-weather-console-dht22-bmp180-ldr-oled.md)
