# 164 - Pico Solar Panel HUD

Build a solar charging diagnostics station that measures battery voltage and panel output current to display live power efficiency metrics on an OLED.

## Goal
Learn how to monitor multiple analog inputs, calculate power metrics (voltage, current, power in milliwatts), and design graphical OLED dashboards in MicroPython.

## What You Will Build
A solar power diagnostic console:
- **Voltage Sensor Divider (GP26)**: Measures battery voltage (0 to 15V).
- **ACS712 Current Sensor (GP27)**: Measures charging current (0 to 5A).
- **SSD1306 OLED (GP4, GP5)**: Displays voltage, current, and calculated charging power in milliwatts ($P = V \times I$).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Voltage Sensor Module | `potentiometer` | Yes (represented by potentiometer) | Yes (resistor divider) |
| ACS712 Current Sensor | `potentiometer` | Yes (represented by potentiometer) | Yes (5A ACS712 module) |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Voltage Sensor | OUT | GP26 | Yellow | Analog input for voltage divider |
| Voltage Sensor | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| ACS712 Sensor | OUT | GP27 | Green | Analog input for current sensor |
| ACS712 Sensor | VCC / GND | 5V / GND | Red / Black | Power lines (requires 5V) |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The ACS712 Hall-effect sensor runs on 5V, so connect its VCC to the VBUS pin. The voltage divider output on GP26 must not exceed 3.3V. The OLED shares the I2C Bus 0 on GP4/GP5.

## Code
```python
from machine import Pin, ADC, I2C
import utime, ssd1306

VOLT_PIN = ADC(26)
ACS_PIN  = ADC(27)

i2c  = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

oled.fill(0)
oled.text("Solar HUD Ready", 4, 28)
oled.show()
utime.sleep(1.0)

print("Solar power diagnosis engine online.")

while True:
    raw_volt = VOLT_PIN.read_u16()
    raw_acs  = ACS_PIN.read_u16()
    
    # 1. Convert voltage: maps 16-bit ADC (0-65535) to 0-15.0V range
    voltage = raw_volt * 15.0 / 65535.0
    
    # 2. Convert current: maps 16-bit ADC (0-65535) to 0-5000mA range
    current_mA = raw_acs * 5000.0 / 65535.0
    
    # 3. Calculate Power (Power = Voltage * Current)
    power_mW = voltage * current_mA
    
    oled.fill(0)
    
    # Draw header block
    oled.fill_rect(0, 0, 128, 14, 1)
    oled.text("SOLAR POWER HUD", 4, 3, 0)  # Inverted text (black on white)
    
    # Display values
    # Row 1: Voltage
    oled.text("Volts: {:.2f} V".format(voltage), 10, 20, 1)
    
    # Row 2: Current
    oled.text("Amps : {:.3f} A".format(current_mA / 1000.0), 10, 34, 1)
    
    # Row 3: Power
    oled.text("Power: {:.1f} mW".format(power_mW), 10, 48, 1)
    
    oled.rect(0, 0, 128, 64, 1)
    oled.show()
    
    print("V:{:.2f}V I:{:.3f}A P:{:.1f}mW".format(voltage, current_mA / 1000.0, power_mW))
    utime.sleep(0.5)  # Update twice per second
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two potentiometers** (for voltage/current), and **SSD1306 OLED** onto the canvas.
2. Connect Voltage pot to **GP26**, Current pot to **GP27**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the potentiometer controls to adjust simulated charging rates and watch the OLED calculate power.

## Expected Output
Terminal:
```
Solar power diagnosis engine online.
V:12.50V I:1.200A P:15000.0mW
V:12.60V I:1.210A P:15246.0mW
```

## Expected Canvas Behavior
* Startup: OLED reads `Volts: 0.00 V` / `Amps: 0.000 A` / `Power: 0.0 mW`.
* Slide Voltage to maximum (GP26 high): OLED reads `Volts: 15.00 V`.
* Slide Current to maximum (GP27 high): OLED reads `Amps: 5.000 A`, and `Power` updates to `75000.0 mW`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `voltage * current_mA` | Multiplies voltage by current in milliamperes to calculate power load in milliwatts. |
| `oled.fill_rect(0, 0, 128, 14, 1)` | Draws a solid white banner on the OLED for the header background. |

## Hardware & Safety Concept: Solar Charge Controller Diagnostics
Diagnostic panels let engineers monitor battery states and panel output in solar charging systems. High voltage or overcurrent can damage batteries, so charge controllers monitor these metrics and automatically open cutoff relays if charging parameters cross safety limits.

## Try This! (Challenges)
1. **Overcharge Latch**: Connect a relay on GP10 and open it to cut power if voltage exceeds 14.2V.
2. **Efficiency Rating**: Add code to calculate and display charging efficiency if you enter a panel reference capacity.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Voltage shows static maximum reading | Divider reference missing | Ensure you are scaling the analog readings correctly in code to map the raw 16-bit ADC value (0-65535) to the correct voltage scale. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [112 - Pico ACS712 Current Monitor LCD](../intermediate/112-pico-acs712-current-monitor-lcd.md)
- [158 - Pico MCP3208 Eight-Channel ADC Logger OLED](158-pico-mcp3208-eight-channel-adc-logger-oled.md)
