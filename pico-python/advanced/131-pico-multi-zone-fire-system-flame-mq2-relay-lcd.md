# 131 - Pico Multi-Zone Fire System Flame MQ-2 Relay LCD

Build a two-zone industrial fire and gas detection system that independently monitors a flame sensor and an MQ-2 gas sensor, triggers zone-specific relay outputs and a siren, and displays alert status on an I2C LCD.

## Goal
Learn how to implement independent multi-zone fire alarm logic using a digital flame sensor and an analog MQ-2 gas sensor, control zone-specific relay outputs, manage a siren alarm, and display zone status on an I2C character LCD in MicroPython.

## What You Will Build
A two-zone fire and gas alarm system:
- **Flame Sensor (GP14)**: Zone 1 — detects fire (LOW output on detection).
- **MQ-2 Gas Sensor (GP26 ADC)**: Zone 2 — detects LPG, smoke, or combustible gas.
- **Relay 1 — Zone 1 Solenoid (GP10)**: Activates Zone 1 fire suppression on flame detection.
- **Relay 2 — Zone 2 Ventilation (GP11)**: Activates Zone 2 ventilation on gas detection.
- **Buzzer (GP15)**: Sounds a multi-pattern siren based on the active zone.
- **I2C 16x2 LCD (GP4, GP5)**: Displays active zone label, sensor readings, and relay states.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Flame Sensor Module | `button` | Yes (digital output) | Yes |
| MQ-2 Gas Sensor | `potentiometer` | Yes (analog input) | Yes |
| Relay Module × 2 | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Flame Sensor | VCC | 3.3V (3V3) | Red | Sensor power |
| Flame Sensor | GND | GND | Black | Ground reference |
| Flame Sensor | DO (Digital Out) | GP14 | Orange | LOW = flame detected |
| MQ-2 Gas Sensor | VCC | 5V (VBUS) | Red | Heater power (5V) |
| MQ-2 Gas Sensor | GND | GND | Black | Ground reference |
| MQ-2 Gas Sensor | AOUT | GP26 | Yellow | Analog gas concentration |
| Relay 1 (Zone 1) | IN | GP10 | Orange | Zone 1 fire suppression control |
| Relay 2 (Zone 2) | IN | GP11 | Blue | Zone 2 ventilation control |
| Active Buzzer | VCC (+) | GP15 | Red | Multi-zone siren output |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the Flame Sensor DO to GP14 (inverted logic — LOW means fire). Connect MQ-2 AOUT to GP26. Connect Relay 1 to GP10 and Relay 2 to GP11. Connect the buzzer to GP15 and the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, ADC, I2C
import utime
from machine_lcd import I2cLcd

flame_sensor = Pin(14, Pin.IN, Pin.PULL_UP)   # LOW = fire
mq2_sensor   = ADC(26)
relay_z1     = Pin(10, Pin.OUT)   # Zone 1 — Fire suppression
relay_z2     = Pin(11, Pin.OUT)   # Zone 2 — Ventilation
buzzer       = Pin(15, Pin.OUT)

# I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

relay_z1.value(0)
relay_z2.value(0)
buzzer.value(0)

GAS_THRESHOLD = 35000  # Raw ADC — above this = gas alarm

lcd.clear()
lcd.putstr("Fire Alarm Sys.")
utime.sleep(1)

print("Multi-zone fire system active.")

while True:
    flame_det = (flame_sensor.value() == 0)  # LOW = fire
    gas_raw   = mq2_sensor.read_u16()
    gas_det   = (gas_raw > GAS_THRESHOLD)

    # Zone 1 — Flame
    relay_z1.value(1 if flame_det else 0)

    # Zone 2 — Gas
    relay_z2.value(1 if gas_det else 0)

    # Determine highest-priority alert
    if flame_det and gas_det:
        zone_label = "Z1+Z2 FIRE+GAS"
    elif flame_det:
        zone_label = "Z1 FIRE ALARM "
    elif gas_det:
        zone_label = "Z2 GAS  ALARM "
    else:
        zone_label = "ALL CLEAR     "

    # LCD
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr(zone_label)
    lcd.move_to(0, 1)
    lcd.putstr("R1:{} R2:{} G:{}".format(
        "ON" if flame_det else "OF",
        "ON" if gas_det   else "OF",
        min(99, int(gas_raw * 99 / 65535))))

    # Siren patterns
    if flame_det and gas_det:
        # Rapid dual alarm
        buzzer.value(1); utime.sleep_ms(100)
        buzzer.value(0); utime.sleep_ms(100)
    elif flame_det:
        # Slow fire alarm pulse
        buzzer.value(1); utime.sleep_ms(400)
        buzzer.value(0); utime.sleep_ms(400)
    elif gas_det:
        # Medium gas alarm pulse
        buzzer.value(1); utime.sleep_ms(200)
        buzzer.value(0); utime.sleep_ms(200)
    else:
        buzzer.value(0)
        utime.sleep_ms(500)

    print("Zone:{} | Flame:{} | Gas raw:{} ({}%)".format(
        zone_label.strip(), flame_det, gas_raw,
        min(99, int(gas_raw * 99 / 65535))))
```

## What to Click in MbedO
1. Drag **Pico**, **Flame Sensor Button**, **Potentiometer** (MQ-2), **two Relays**, **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect Flame Sensor to **GP14**, MQ-2 to **GP26**, Relay 1 to **GP10**, Relay 2 to **GP11**, Buzzer to **GP15**, and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the Flame Sensor button to trigger Zone 1. Slide the MQ-2 slider above the threshold to trigger Zone 2. Trigger both simultaneously for the combined alarm.

## Expected Output
```
Multi-zone fire system active.
Zone:ALL CLEAR | Flame:False | Gas raw:5000 (7%)
Zone:Z1 FIRE ALARM | Flame:True | Gas raw:5000 (7%)
Zone:Z1+Z2 FIRE+GAS | Flame:True | Gas raw:40000 (60%)
```

## Expected Canvas Behavior
- Relay 1 closes when the Flame button is clicked. Relay 2 closes when the MQ-2 slider exceeds the threshold. The buzzer pulse rate varies depending on the active alarm combination. The LCD shows the combined zone status.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `flame_sensor.value() == 0` | Inverted logic: the KY-026 flame sensor outputs LOW when fire is detected (active-low output). |
| Siren patterns `if flame_det and gas_det` | Uses the most urgent pattern (rapid beeping) when both zones are simultaneously alarming. |

## Hardware & Safety Concept: Zone Independence in Fire Panels
Professional BS5839-certified fire alarm panels treat each zone as an independent circuit. A fault or alarm in Zone 1 must never suppress or mask Zone 2. Each zone has its own supervised wire loop, relay output, and LED indicator. This project implements software zone independence — each relay is driven by its own sensor logic.

## Try This! (Challenges)
1. **Alarm Latching**: Modify the code so each zone relay latches ON once triggered and requires a physical button press to reset (as required by fire codes).
2. **Event Log**: Use a circular buffer to store the last 5 alarm events (zone, timestamp ticks) and display them on the LCD with a button press.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Flame zone always triggered | Ambient light hitting sensor | Point flame sensor away from direct sunlight or incandescent sources. Adjust onboard sensitivity pot. |
| Gas zone always triggered | Threshold too low for clean air | Print MQ-2 raw values in clean air and set `GAS_THRESHOLD` above the baseline. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [120 - Pico Multi-Sensor Alarm System LCD](../intermediate/120-pico-multi-sensor-alarm-system-lcd.md)
- [71 - Pico Flame Sensor Alarm](../intermediate/71-pico-flame-sensor-alarm.md)
- [72 - Pico MQ-2 Gas Alarm](../intermediate/72-pico-mq2-gas-alarm.md)
