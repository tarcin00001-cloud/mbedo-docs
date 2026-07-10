# 79 - Pico Soil Irrigator LCD

Build an automated crop watering station that displays soil moisture percentages on an I2C LCD and controls a watering valve relay.

## Goal
Learn how to monitor analog soil moisture sensors, implement dual-threshold hysteresis control logic, and output status text and percentages to an I2C character LCD in MicroPython.

## What You Will Build
An irrigation control panel:
- **Soil Moisture Sensor (GP26)**: Measures soil moisture.
- **Relay Module (GP15)**: Switches power ON to a water solenoid valve when soil is dry, and OFF when watered.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the calculated soil moisture percentage and active valve status ("WATERING" or "STANDBY").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Soil Moisture Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Relay Module | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Soil Sensor | VCC (+) | 3.3V (3V3) | Red | Power line |
| Soil Sensor | GND (−) | GND | Black | Ground reference |
| Soil Sensor | OUT (Analog) | GP26 | Yellow | Analog input signal |
| Relay Module | IN | GP15 | Orange | Relay control signal (HIGH on activate) |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the soil moisture sensor output to GP26, and the relay control input pin to GP15. Connect the LCD to the GP4/GP5 pins. All ground connections are shared.

## Code
```python
from machine import Pin, ADC, I2C
import utime
from machine_lcd import I2cLcd

soil_sensor = ADC(26) # GP26 = ADC channel 0
valve_relay = Pin(15, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Calibration bounds (raw ADC scale 0 - 65535, high = dry)
DRY_VALUE = 48000
WET_VALUE = 12000

# Control limits
DRY_LIMIT  = 38000 # Turn valve ON if reading > 38000 (Soil is dry)
WET_LIMIT  = 22000 # Turn valve OFF if reading < 22000 (Soil is watered)

valve_relay.value(0) # Start closed (OFF)
watering_active = False

lcd.clear()
lcd.putstr("Irrigation Panel")
lcd.move_to(0, 1)
lcd.putstr("Initializing...")
utime.sleep(1)

while True:
    raw = soil_sensor.read_u16()
    
    # Calculate soil moisture percentage (invert because wet soil has lower voltage/reading)
    percent = (DRY_VALUE - raw) * 100 // (DRY_VALUE - WET_VALUE)
    if percent < 0:
        percent = 0
    elif percent > 100:
        percent = 100
        
    # Hysteresis control logic
    if raw > DRY_LIMIT and not watering_active:
        watering_active = True
        valve_relay.value(1) # Open water valve
        print(">> Solenoid Valve Open")
    elif raw < WET_LIMIT and watering_active:
        watering_active = False
        valve_relay.value(0) # Close water valve
        print(">> Solenoid Valve Closed")
        
    # Update LCD Display
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("Moisture: " + str(percent) + " %")
    lcd.move_to(0, 1)
    lcd.putstr("Valve: " + ("WATERING" if watering_active else "STANDBY"))
    
    print("Moisture:", percent, "% | Valve:", "OPEN" if watering_active else "CLOSED")
    utime.sleep(1) # 1-second update cycle
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Soil Moisture Sensor** (represented by potentiometer), **Relay**, and **I2C LCD** onto the canvas.
2. Connect Sensor to **GP26**, Relay to **GP15**, and LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the soil moisture sensor to maximum (dry) to see the relay activate and LCD status update.

## Expected Output
```
Moisture: 0 % | Valve: CLOSED
Moisture: 10 % | Valve: CLOSED
>> Solenoid Valve Open
Moisture: 85 % | Valve: OPEN
>> Solenoid Valve Closed
```
(On screen: "Moisture: X %" and "Valve: WATERING/STANDBY" on respective lines.)

## Expected Canvas Behavior
- The relay component clicks active and the LCD text updates to show "WATERING" when the soil moisture sensor slider is moved past 38000.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `(DRY_VALUE - raw) * 100 // ...` | Scales and inverts the raw analog value to calculate a soil moisture percentage. |
| `valve_relay.value(1)` | Drives GP15 HIGH to open the water solenoid valve. |

## Hardware & Safety Concept: Sensor Fail-Safe Protection
Soil moisture sensors placed in active dirt are exposed to moisture, fertilizer acids, and mechanical stress. Sensor failure (e.g. broken signal wire) typically causes the analog reading to drift high (dry soil state). To prevent the system from watering indefinitely on sensor failure, smart irrigation systems implement a **maximum runtime water cutoff** safety loop.

## Try This! (Challenges)
1. **Critical High Alert**: Connect a buzzer on GP14 and sound a rapid warning beep if the moisture levels fall below 15%.
2. **Override Switch**: Connect a button on GP13 to manually override the system and turn the water valve ON/OFF.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD displays incorrect moisture | Calibration limits | Measure raw ADC values when dry and wet, and update `DRY_VALUE` and `WET_VALUE` accordingly. |
| Valve cycles rapidly | Hysteresis too narrow | Expand the gap between `DRY_LIMIT` and `WET_LIMIT` to prevent sensor noise from chattering the relay. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [42 - Pico Soil Moisture Serial](42-pico-soil-moisture-serial.md)
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [68 - Pico Soil Irrigator Relay](68-pico-soil-irrigator-relay.md)
