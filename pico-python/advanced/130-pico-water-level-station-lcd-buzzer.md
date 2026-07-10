# 130 - Pico Water Level Station LCD Buzzer

Build a four-level water tank monitoring station that reads a water level sensor, classifies the fill level into four zones (Empty, Low, Normal, Full), triggers a buzzer alarm at overflow, and displays the level and zone on an I2C LCD.

## Goal
Learn how to read an analog water level sensor, map ADC readings to named water zones, produce zone-specific buzzer alerts, and display live level readings on an I2C character LCD in MicroPython.

## What You Will Build
A water tank monitoring station:
- **Water Level Sensor (GP26)**: Reads current tank water level as an analog voltage.
- **Active Buzzer (GP15)**: Alerts on overflow (Full zone) and empty (Empty zone).
- **LED (GP13)**: ON during alarm states.
- **I2C 16x2 LCD (GP4, GP5)**: Displays measured level, percentage, and zone label.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Water Level Sensor | `potentiometer` | Yes (slider represents level) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| LED | `led` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Water Level Sensor | VCC (+) | 3.3V (3V3) | Red | Sensor power |
| Water Level Sensor | GND (−) | GND | Black | Ground reference |
| Water Level Sensor | OUT (Analog) | GP26 | Yellow | Analog level output |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Shared return path to ground |
| LED | Anode (+, via 330 Ω) | GP13 | Orange | Alarm indicator |
| LED | Cathode (−) | GND | Black | Ground return |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the water level sensor output to GP26. Connect the buzzer to GP15 and the LED to GP13 (through 330 Ω). Connect the LCD to GP4/GP5. All grounds are shared. In a real installation, the sensor probes are placed vertically in the tank.

## Code
```python
from machine import Pin, ADC, I2C
import utime
from machine_lcd import I2cLcd

level_sensor = ADC(26)  # GP26 = ADC channel 0
buzzer       = Pin(15, Pin.OUT)
alarm_led    = Pin(13, Pin.OUT)

# I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

buzzer.value(0)
alarm_led.value(0)

# Zone thresholds (raw ADC 0–65535)
# Higher raw value = higher water level
ZONE_EMPTY  = 10000   # Below this = EMPTY
ZONE_LOW    = 25000   # Below this = LOW
ZONE_NORMAL = 50000   # Below this = NORMAL
                      # At or above = FULL

def classify_zone(raw):
    if raw < ZONE_EMPTY:
        return "EMPTY ", True    # (label, alarm)
    elif raw < ZONE_LOW:
        return "LOW   ", False
    elif raw < ZONE_NORMAL:
        return "NORMAL", False
    else:
        return "FULL  ", True    # Overflow alarm

def raw_to_percent(raw):
    return min(100, int(raw * 100 / 65535))

lcd.clear()
lcd.putstr("Water Level Stn")
utime.sleep(1)

print("Water level station active.")

while True:
    raw     = level_sensor.read_u16()
    pct     = raw_to_percent(raw)
    zone, alarm = classify_zone(raw)

    # Bar graph characters (16-char wide, 0-100%)
    bar_len = bar_len = min(16, int(pct * 16 / 100))
    bar     = "=" * bar_len + "-" * (16 - bar_len)

    # LCD update
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("{:3d}% Zone:{}".format(pct, zone))
    lcd.move_to(0, 1)
    lcd.putstr(bar)

    # Alarm control
    if alarm:
        alarm_led.value(1)
        buzzer.value(1)
        utime.sleep_ms(300)
        buzzer.value(0)
        utime.sleep_ms(300)
    else:
        alarm_led.value(0)
        buzzer.value(0)
        utime.sleep_ms(600)

    print("Level: {:3d}% | Zone: {} | Raw: {}".format(pct, zone.strip(), raw))
```

## What to Click in MbedO
1. Drag **Pico**, **Potentiometer** (water level sensor), **Active Buzzer**, **LED**, and **I2C LCD** onto the canvas.
2. Connect Potentiometer to **GP26**, Buzzer to **GP15**, LED to **GP13**, and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the potentiometer from 0 to maximum. Observe the zone label and bar graph update. Alarm activates at EMPTY and FULL extremes.

## Expected Output
```
Water level station active.
Level:   0% | Zone: EMPTY | Raw: 4200
Level:  45% | Zone: NORMAL | Raw: 29500
Level: 100% | Zone: FULL | Raw: 65200
```

## Expected Canvas Behavior
- The LCD bar graph grows as the potentiometer slider is raised. Zone label changes from EMPTY → LOW → NORMAL → FULL. The buzzer and LED pulse at EMPTY and FULL.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `classify_zone(raw)` | Returns a tuple of (zone label string, alarm boolean) based on defined ADC thresholds. |
| `"=" * bar_len + "-" * (16 - bar_len)` | Builds a 16-character text bar graph, filled proportionally to the measured percentage. |

## Hardware & Safety Concept: Resistive Water Level Sensors
Resistive water level sensors work by measuring the electrical conductivity between exposed metal probes submerged in water. As more probes are submerged, conductivity increases and analog output voltage rises. Because tap water conductivity varies with mineral content, calibrate the zone thresholds by printing raw values at known fill levels.

## Try This! (Challenges)
1. **Pump Auto-Control**: Connect a relay on GP12 that opens a fill valve when the EMPTY zone is detected and closes it when NORMAL is reached (using hysteresis).
2. **OLED Graphical Tank**: Display a graphical tank silhouette on an SSD1306 OLED with a fill rectangle rising proportionally.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Zone never reaches FULL | Threshold set too high | Print raw ADC values at maximum fill and adjust `ZONE_NORMAL` accordingly. |
| Alarm always ON at boot | EMPTY threshold too high | At power-on, the tank may be at a low level. Verify `ZONE_EMPTY` is below the idle sensor value. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [104 - Pico Water Overflow Alarm](../intermediate/104-pico-water-overflow-alarm.md)
- [122 - Pico Greenhouse Controller DHT22 Relay LCD](122-pico-greenhouse-controller-dht22-relay-lcd.md)
