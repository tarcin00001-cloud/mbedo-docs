# 120 - Pico Multi-Sensor Alarm System LCD

Build a four-zone alarm system that monitors a flame sensor, MQ-2 gas sensor, PIR motion sensor, and door reed switch simultaneously, triggering a buzzer and displaying the active alarm zone on an I2C LCD.

## Goal
Learn how to monitor multiple digital and analog sensors simultaneously in a single main loop, implement latching alarm states for multiple zones, and display the alarming zone on an I2C character LCD in MicroPython.

## What You Will Build
A four-zone home alarm system:
- **Flame Sensor (GP14)**: Detects fire (LOW output on detection).
- **MQ-2 Gas Sensor (GP26 ADC)**: Detects gas or smoke above a threshold.
- **PIR Sensor (GP13)**: Detects motion (HIGH output on detection).
- **Door Reed Switch (GP12)**: Detects door open/closed state.
- **Buzzer (GP15)**: Sounds a pulsing alarm when any zone triggers.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the active alarm zone and state.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Flame Sensor Module | `button` | Yes (digital output, represented by button) | Yes |
| MQ-2 Gas Sensor | `potentiometer` | Yes (analog ADC input) | Yes |
| PIR Motion Sensor | `button` | Yes (digital output, represented by button) | Yes |
| Reed Switch (door sensor) | `button` | Yes (represented by button) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Reed Switch | Signal | GP12 | White | Door sensor (LOW = closed, HIGH = open) |
| PIR Sensor | OUT | GP13 | Blue | Motion detector output (HIGH on motion) |
| Flame Sensor | OUT | GP14 | Orange | Flame indicator (LOW on flame detected) |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm siren output |
| Reed / PIR / Flame | GND | GND | Black | Ground reference |
| MQ-2 Gas Sensor | VCC | 5V (VBUS) | Red | Sensor heater power (5V) |
| MQ-2 Gas Sensor | GND | GND | Black | Ground reference |
| MQ-2 Gas Sensor | AOUT | GP26 | Yellow | Analog gas concentration output |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** All digital sensors (Reed, PIR, Flame) output logic levels to GPIO pins. The MQ-2 analog output connects to GP26. Connect the buzzer to GP15 and the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, ADC, I2C
import utime
from machine_lcd import I2cLcd

door_sensor   = Pin(12, Pin.IN, Pin.PULL_DOWN)
pir_sensor    = Pin(13, Pin.IN, Pin.PULL_DOWN)
flame_sensor  = Pin(14, Pin.IN, Pin.PULL_UP) # LOW = flame detected
mq2_sensor    = ADC(26)
buzzer        = Pin(15, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# MQ-2 gas threshold (raw ADC 0 - 65535; high = more gas)
GAS_THRESHOLD = 35000

alarm_state  = False
alarm_zone   = "NONE"
buzzer.value(0)

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Security System")
lcd.move_to(0, 1)
lcd.putstr("PIR Warming...")
utime.sleep(2) # Allow PIR to stabilize

print("Multi-zone alarm system active.")

while True:
    # Read all sensors
    door_open   = (door_sensor.value() == 1)
    pir_motion  = (pir_sensor.value() == 1)
    flame_det   = (flame_sensor.value() == 0) # LOW = fire
    gas_raw     = mq2_sensor.read_u16()
    gas_high    = (gas_raw > GAS_THRESHOLD)
    
    # Determine active zone (priority: fire > gas > motion > door)
    if flame_det:
        alarm_zone  = "FIRE DETECTED"
        alarm_state = True
    elif gas_high:
        alarm_zone  = "GAS DETECTED"
        alarm_state = True
    elif pir_motion:
        alarm_zone  = "MOTION DETECT"
        alarm_state = True
    elif door_open:
        alarm_zone  = "DOOR OPEN"
        alarm_state = True
    else:
        alarm_zone  = "ALL CLEAR"
        alarm_state = False
        
    # Update LCD
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("Zone: " + alarm_zone[:10])
    lcd.move_to(0, 1)
    if alarm_state:
        lcd.putstr("!! ALARM !!")
    else:
        lcd.putstr("System: ARMED")
        
    # Pulsing buzzer during alarm
    if alarm_state:
        buzzer.value(1)
        utime.sleep_ms(250)
        buzzer.value(0)
        utime.sleep_ms(250)
    else:
        utime.sleep_ms(500)
        
    print("Zone:{} | Gas:{} | Alarm:{}".format(
        alarm_zone, gas_raw, alarm_state))
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **three Buttons** (Flame, PIR, Door), **Potentiometer** (MQ-2), **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect Flame Button to **GP14**, PIR Button to **GP13**, Door Button to **GP12**, Potentiometer to **GP26**, Buzzer to **GP15**, and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click each button or raise the potentiometer to trigger the alarm zones and observe the LCD.

## Expected Output
```
Multi-zone alarm system active.
Zone:ALL CLEAR | Gas:4500 | Alarm:False
Zone:FIRE DETECTED | Gas:4500 | Alarm:True
Zone:GAS DETECTED | Gas:38000 | Alarm:True
```

## Expected Canvas Behavior
- Clicking the Flame button triggers the "FIRE DETECTED" zone. Raising the potentiometer slider above the GAS_THRESHOLD triggers "GAS DETECTED". The LCD updates with the active zone and the buzzer component pulses ON/OFF.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `flame_sensor.value() == 0` | Detects flame because the KY-026 flame sensor module outputs LOW when it detects infrared radiation from fire. |
| Priority `if/elif` chain | Ensures higher-priority alarms (fire, gas) are displayed over lower-priority ones (motion, door). |

## Hardware & Safety Concept: Alarm Zone Priority Encoding
Professional alarm panels assign numeric priority levels to each zone. Fire and CO alarms always override intrusion alarms on the main siren and control panel display. This is a safety-critical design requirement: a burglar alarm must never suppress a simultaneous fire alarm on the display.

## Try This! (Challenges)
1. **Alarm Latch**: Modify the code so the alarm latches ON (requires a button reset) instead of self-clearing when the sensor returns to normal.
2. **Zone Log**: Keep a list of the last 5 triggered zones and display them on the OLED on button press for an event history.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Gas alarm triggers at idle | Threshold set too low | Print MQ-2 raw values in clean air and set `GAS_THRESHOLD` above that baseline level. |
| Flame sensor always triggered | Ambient sunlight or incandescent light | Point flame sensor away from direct light sources. Adjust sensitivity potentiometer on the sensor module. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [71 - Pico Flame Sensor Alarm](71-pico-flame-sensor-alarm.md)
- [72 - Pico MQ-2 Gas Alarm](72-pico-mq2-gas-alarm.md)
- [74 - Pico Motion Security PIR Alarm](74-pico-motion-security-pir-alarm.md)
- [103 - Pico Temperature Alarm HVAC](103-pico-temperature-alarm-hvac.md)
