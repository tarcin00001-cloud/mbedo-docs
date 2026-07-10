# 195 - Pico Medical Cold Chain Monitor

Build a high-security vaccine temperature tracker that monitors a DS18B20 probe, controls cooling compressor relays, and latches a critical alarm state if temperatures deviate from comfort limits.

## Goal
Learn how to read DS18B20 One-Wire sensors, calculate temperature extremes, and implement software-latching alarms that persist even after environmental conditions return to normal.

## What You Will Build
A medical cold-chain temperature console:
- **DS18B20 Temperature Sensor (GP14)**: Measures the internal vaccine box temperature.
- **Relay Module (GP10)**: Controls the cooling compressor (turned ON if temperature rises).
- **Alarm Buzzer (GP11)**: Emits a continuous warning siren during thermal breaches.
- **Alarm Reset Button (GP13)**: Unlatches the warning state after inspection.
- **16x2 I2C LCD (GP4, GP5)**: Displays current temperature, historical min/max values, and latch warnings.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DS18B20 Temperature Probe | `ds18b20` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 4.7k-ohm Resistor | `resistor` | Optional | Yes (One-Wire pull-up) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS18B20 | DATA | GP14 | Yellow | One-Wire sensor data (requires 4.7k pull-up) |
| DS18B20 | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| Relay Module | IN | GP10 | Orange | Cooling compressor switch |
| Active Buzzer | VCC (+) | GP11 | Blue | Audible alarm output |
| Reset Button | Terminal 1 | GP13 | White | Alarm disarm switch |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The DS18B20 data line requires a 4.7k-ohm pull-up resistor to 3.3V. Connect the sensor data line to GP14. The LCD shares GP4/GP5.

## Code
```python
from machine import Pin, I2C
import utime, onewire, ds18x20
from machine_lcd import I2cLcd

# Pin definitions
ow_pin = Pin(14)
relay = Pin(10, Pin.OUT)
buzzer = Pin(11, Pin.OUT)
btn_reset = Pin(13, Pin.IN, Pin.PULL_UP)

relay.value(0)
buzzer.value(0)

# Setup DS18B20
ow = onewire.OneWire(ow_pin)
ds = ds18x20.DS18X20(ow)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Medical Cold Chain boundaries (typical vaccine fridge: 2°C to 8°C)
TEMP_MIN_LIMIT = 2.0
TEMP_MAX_LIMIT = 8.0

# Memory variables
min_temp = 100.0
max_temp = -100.0
alarm_latched = False
first_read = True

# Search for DS18B20 sensors
roms = ds.scan()
print("Found One-Wire devices:", roms)

lcd.clear()
lcd.putstr("Cold Chain Mon\nDevice Armed")
utime.sleep(1.5)

while True:
    now = utime.ticks_ms()
    
    # 1. Read temperature from DS18B20
    temp = 0.0
    sensor_ok = False
    if roms:
        try:
            ds.convert_temp()
            utime.sleep_ms(750)  # Wait for conversion (12-bit requires 750ms)
            temp = ds.read_temp(roms[0])
            sensor_ok = True
        except Exception:
            pass
            
    # 2. Update historical limits
    if sensor_ok:
        if first_read:
            min_temp = temp
            max_temp = temp
            first_read = False
        else:
            min_temp = min(min_temp, temp)
            max_temp = max(max_temp, temp)
            
        # 3. Evaluate safety limits and latch alarm
        if (temp < TEMP_MIN_LIMIT or temp > TEMP_MAX_LIMIT) and not alarm_latched:
            alarm_latched = True
            print("!! ALERT: TEMPERATURE BREACH DETECTED !!")
            
    # 4. Handle cooling compressor relay actuation (automatic cooling)
    if sensor_ok and temp > TEMP_MAX_LIMIT:
        relay.value(1)  # Turn compressor ON
        compressor_state = "ON "
    else:
        relay.value(0)  # Turn compressor OFF
        compressor_state = "OFF"
        
    # 5. Alarm feedback
    if alarm_latched:
        # Siren pulse
        buzzer.value(now // 500 % 2)
        
        # Check for manual alarm reset button click (GP13)
        if btn_reset.value() == 0:
            alarm_latched = False
            # Reset historical tracking bounds
            min_temp = temp
            max_temp = temp
            buzzer.value(0)
            print("Alarm unlatched. Re-armed.")
            lcd.clear(); lcd.putstr("Alarm Reset\nRe-arming...")
            utime.sleep(1.0)
    else:
        buzzer.value(0)
        
    # 6. Update LCD Display
    lcd.clear()
    if alarm_latched:
        lcd.putstr("BREACH! T:{:.1f}C\n".format(temp))
        lcd.putstr("MIN:{:.1f} MAX:{:.1f}".format(min_temp, max_temp))
    else:
        if sensor_ok:
            lcd.putstr("Temp: {:.1f}C {}\n".format(temp, compressor_state))
            lcd.putstr("Min:{:.1f} Max:{:.1f}".format(min_temp, max_temp))
        else:
            lcd.putstr("SENSOR ERROR\nSystem Idle")
            
    if sensor_ok:
        print("Temp: {:.2f}C | Min: {:.1f} | Max: {:.1f} | Alarm: {}".format(
            temp, min_temp, max_temp, "LATCHED" if alarm_latched else "OK"
        ))
        
    utime.sleep_ms(250)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DS18B20**, **Relay**, **Active Buzzer**, **Push Button**, and **I2C LCD** onto the canvas.
2. Connect DS18B20 to **GP14**, Relay to **GP10**, Buzzer to **GP11**, Button to **GP13**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the DS18B20 temperature slider above 8°C. Verify that the compressor relay closes and the alarm buzzer begins to beep.
5. Slide the temperature back to 5°C. Observe that the buzzer continues to beep and the display still reads `BREACH!` (latched alert). Press the reset button on GP13 to silence it.

## Expected Output
Terminal:
```
Found One-Wire devices: [bytearray(b'(\x00\x00\x00\x00\x00\x00\x00')]
Temp: 5.40C | Min: 5.4 | Max: 5.4 | Alarm: OK
!! ALERT: TEMPERATURE BREACH DETECTED !!
Temp: 9.20C | Min: 5.4 | Max: 9.2 | Alarm: LATCHED
Temp: 5.10C | Min: 5.1 | Max: 9.2 | Alarm: LATCHED
Alarm unlatched. Re-armed.
```

## Expected Canvas Behavior
* Normal state (temp 5.0°C): Relay is OFF. Buzzer is silent. LCD reads `Min:5.4 Max:5.4`.
* Slide temperature to 9.0°C: Relay 1 turns ON. Buzzer pulses. LCD reads `BREACH!`.
* Return temperature to 5.0°C: Relay 1 turns OFF. Buzzer keeps pulsing. LCD keeps reading `BREACH!`.
* Click Button (GP13): Buzzer stops. LCD displays normal temperature stats.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `ds.convert_temp()` | Commands the DS18B20 to start an analog-to-digital temperature conversion. |
| `ds.read_temp(roms[0])` | Reads the scratchpad memory of the target DS18B20 probe to retrieve the temperature. |

## Hardware & Safety Concept: Alarm Latching in Critical Logistics
In medical cold-chain logistics, if vaccine vials are exposed to room temperature for even one hour, they lose potency and must be discarded. If a monitor alarm was not latched, a temporary cooling failure (e.g. unplugging a freezer box for an hour) would go unnoticed once the temperature returned to normal. Latching the alarm state ensures that inspectors are notified of any thermal violations that occurred during transit.

## Try This! (Challenges)
1. **Critical Freeze Alert**: Sound a fast yelp alarm if the temperature falls below 0°C, risking freezing the vaccines.
2. **Periodic Flash Log**: Write the min and max temperatures to the Pico's internal flash memory every hour.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor not detected / error | Missing pull-up resistor | One-Wire bus structures require a 4.7k-ohm pull-up resistor between the DATA line (GP14) and 3.3V. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [123 - Pico Cold Chain Monitor DS18B20 Relay LCD](../advanced/123-pico-cold-chain-monitor-ds18b20-relay-lcd.md)
- [136 - Pico DS18B20 Multi-Probe Temperature Network LCD](../advanced/136-pico-ds18b20-multi-probe-temperature-network-lcd.md)
