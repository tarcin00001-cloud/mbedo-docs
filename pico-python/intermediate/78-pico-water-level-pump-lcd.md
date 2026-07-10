# 78 - Pico Water Level Pump LCD

Build a sump pump control panel that displays water fill percentages on an I2C LCD and controls a pump relay.

## Goal
Learn how to monitor analog liquid level sensors, implement dual-threshold hysteresis control logic, and output status text and percentages to an I2C character LCD in MicroPython.

## What You Will Build
A sump pump dashboard:
- **Water Level Sensor (GP26)**: Measures water depth.
- **Relay Module (GP15)**: Switches power ON to a water pump when water is high, and OFF when low.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the calculated depth percentage and active pump status ("RUNNING" or "STANDBY").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Water Level Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Relay Module | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Water Level Sensor | VCC (+) | 3.3V (3V3) | Red | Power line |
| Water Level Sensor | GND (−) | GND | Black | Ground reference |
| Water Level Sensor | OUT (Analog) | GP26 | Yellow | Analog input signal |
| Relay Module | IN | GP15 | Orange | Relay control signal (HIGH on activate) |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the water level sensor output to GP26, and the relay control input pin to GP15. Connect the LCD to the GP4/GP5 pins. All ground connections are shared.

## Code
```python
from machine import Pin, ADC, I2C
import utime
from machine_lcd import I2cLcd

water_sensor = ADC(26) # GP26 = ADC channel 0
pump_relay   = Pin(15, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Calibration bounds (raw ADC scale 0 - 65535)
SENSOR_MIN = 8000
SENSOR_MAX = 52000

# Control limits
PUMP_ON_LIMIT  = 35000 # Turn pump ON if water level > 35000
PUMP_OFF_LIMIT = 15000 # Turn pump OFF if water level < 15000

pump_relay.value(0) # Start OFF
pump_state = False

lcd.clear()
lcd.putstr("Sump Pump Panel")
lcd.move_to(0, 1)
lcd.putstr("Initializing...")
utime.sleep(1)

while True:
    raw = water_sensor.read_u16()
    
    # Calculate depth percentage
    percent = (raw - SENSOR_MIN) * 100 // (SENSOR_MAX - SENSOR_MIN)
    if percent < 0:
        percent = 0
    elif percent > 100:
        percent = 100
        
    # Dual-threshold control logic
    if raw > PUMP_ON_LIMIT and not pump_state:
        pump_state = True
        pump_relay.value(1) # Turn pump ON
        print(">> Pump Activated")
    elif raw < PUMP_OFF_LIMIT and pump_state:
        pump_state = False
        pump_relay.value(0) # Turn pump OFF
        print(">> Pump Deactivated")
        
    # Update LCD Display
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("Water Lvl: " + str(percent) + " %")
    lcd.move_to(0, 1)
    lcd.putstr("Pump: " + ("RUNNING" if pump_state else "STANDBY"))
    
    print("Level:", percent, "% | Pump:", "ON" if pump_state else "OFF")
    utime.sleep(1) # 1-second update cycle
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Water Level Sensor** (represented by potentiometer), **Relay**, and **I2C LCD** onto the canvas.
2. Connect Sensor to **GP26**, Relay to **GP15**, and LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the water sensor to maximum (flood) to see the relay activate and LCD status update.

## Expected Output
```
Level: 0 % | Pump: OFF
Level: 75 % | Pump: OFF
>> Pump Activated
Level: 10 % | Pump: ON
>> Pump Deactivated
```
(On screen: "Water Lvl: X %" and "Pump: RUNNING/STANDBY" on respective lines.)

## Expected Canvas Behavior
- The relay component clicks active and the LCD text updates to show "RUNNING" when the water sensor slider is moved past 35000.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `(raw - SENSOR_MIN) * 100 // ...` | Scales the raw analog value into a depth percentage based on the sensor calibration limits. |
| `pump_relay.value(1)` | Drives GP15 HIGH to close the relay contacts, activating the pump. |

## Hardware & Safety Concept: Sensor Scaling Calibration
Because different liquid level sensors have varying electrical properties (resistance/capacitance), dry and wet readings will change based on water mineral content and probe length. Always measure and calibrate dry (`SENSOR_MIN`) and fully submerged (`SENSOR_MAX`) values in your system environment before deployment to ensure accurate readings.

## Try This! (Challenges)
1. **Critical Overflow Alarm**: Connect a buzzer on GP14 and sound a rapid warning beep if the level exceeds 90%.
2. **Override Toggle Switch**: Connect a button on GP13 to manually override the system and turn the pump ON/OFF.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD displays incorrect depth | Calibration limits | Measure raw ADC values when dry and wet, and update `SENSOR_MIN` and `SENSOR_MAX` accordingly. |
| Pump cycles rapidly | Hysteresis too narrow | Expand the gap between `PUMP_ON_LIMIT` and `PUMP_OFF_LIMIT` to prevent sensor noise from chattering the relay. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [41 - Pico Water Level Serial](41-pico-water-level-serial.md)
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [67 - Pico Water Level Pump Relay](67-pico-water-level-pump-relay.md)
