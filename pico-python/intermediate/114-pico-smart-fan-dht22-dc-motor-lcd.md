# 114 - Pico Smart Fan DHT22 DC Motor LCD

Build a temperature-controlled smart fan that automatically ramps DC motor speed based on ambient temperature, and displays the temperature and fan speed on an I2C LCD.

## Goal
Learn how to read a DHT22 temperature sensor, map temperature ranges to PWM duty cycles, control an L298N DC motor driver at variable speeds, and display status on an I2C character LCD in MicroPython.

## What You Will Build
A smart ventilation fan:
- **DHT22 Sensor (GP16)**: Reads ambient temperature.
- **L298N Motor Driver + DC Fan Motor (GP10-12)**: Runs the fan at speed proportional to temperature.
- **I2C 16x2 LCD (GP4, GP5)**: Displays temperature reading and current fan speed percentage.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| DC Motor (fan) | `dc_motor` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | VCC (+) | 3.3V (3V3) | Red | Power line |
| DHT22 | GND (−) | GND | Black | Ground reference |
| DHT22 | DATA | GP16 | Yellow | 1-Wire data signal |
| L298N Driver | ENA | GP10 | Orange | PWM speed control signal |
| L298N Driver | IN1 | GP11 | Yellow | Motor direction pin A |
| L298N Driver | IN2 | GP12 | Green | Motor direction pin B |
| L298N Driver | VCC / GND | 5V (VBUS) / GND | Red / Black | Driver power supply |
| DC Motor | Motor Terminals | OUT1 / OUT2 | — | Driver motor outputs |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the DHT22 DATA pin to GP16. Connect the L298N ENA to GP10 (PWM), IN1 to GP11, and IN2 to GP12. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, PWM, I2C
import utime
import dht
from machine_lcd import I2cLcd

# Initialize DHT22 sensor on GP16
sensor = dht.DHT22(Pin(16))

# Set up L298N motor driver
ena = PWM(Pin(10))
ena.freq(1000)
in1 = Pin(11, Pin.OUT)
in2 = Pin(12, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Fan temperature range: OFF below 22°C, FULL at 38°C
TEMP_OFF  = 22.0
TEMP_FULL = 38.0

# Set motor to forward direction (fan always spins one way)
in1.value(1)
in2.value(0)
ena.duty_u16(0) # Start OFF

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Smart Fan Panel")
lcd.move_to(0, 1)
lcd.putstr("Initializing...")
utime.sleep(2)

print("Smart fan controller active.")

while True:
    try:
        sensor.measure()
        temp_c = sensor.temperature()
        
        # Calculate fan speed duty cycle based on temperature range
        if temp_c <= TEMP_OFF:
            duty    = 0
            percent = 0
        elif temp_c >= TEMP_FULL:
            duty    = 65535
            percent = 100
        else:
            # Linear interpolation between 0% and 100%
            ratio   = (temp_c - TEMP_OFF) / (TEMP_FULL - TEMP_OFF)
            duty    = int(ratio * 65535)
            percent = int(ratio * 100)
            
        ena.duty_u16(duty)
        
        # Update LCD Display
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Temp: {:.1f} C".format(temp_c))
        lcd.move_to(0, 1)
        lcd.putstr("Fan:  {:3d} %".format(percent))
        
        print("Temp: {:.1f} C | Fan: {} %".format(temp_c, percent))
        
    except OSError as e:
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Sensor Error!")
        print(">> Sensor read error:", e)
        
    utime.sleep(2)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22 Sensor**, **L298N Driver**, **DC Motor**, and **I2C LCD** onto the canvas.
2. Connect DHT22 DATA to **GP16**, L298N to **GP10-12**, and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Adjust the DHT22 temperature slider above 22°C and observe the DC motor and LCD respond.

## Expected Output
```
Smart fan controller active.
Temp: 24.5 C | Fan: 16 %
Temp: 35.0 C | Fan: 81 %
```
(On screen: "Temp: 24.5 C" and "Fan: 16 %" on respective lines.)

## Expected Canvas Behavior
- The DC motor component on the canvas spins faster as the DHT22 temperature slider is increased, with the LCD displaying the calculated fan speed percentage.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `(temp_c - TEMP_OFF) / (TEMP_FULL - TEMP_OFF)` | Calculates a 0.0 to 1.0 ratio representing the position within the active temperature range. |
| `int(ratio * 65535)` | Scales the ratio to the full 16-bit PWM duty cycle range. |

## Hardware & Safety Concept: Proportional Temperature Control
This fan uses **proportional control** (P-controller): the output (fan speed) is linearly proportional to the error (temperature above the setpoint). Unlike a simple ON/OFF thermostat, proportional control avoids relay chattering and provides smooth, energy-efficient operation.

## Try This! (Challenges)
1. **Hysteresis Band**: Add a dead band of ±1°C around the OFF threshold to prevent the fan from rapidly switching ON and OFF at the boundary.
2. **Max Speed Alarm**: Connect a buzzer on GP13 that sounds if the fan runs at 100% for more than 60 seconds, indicating a cooling emergency.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor hums but does not spin | PWM duty too low at startup | DC motors have a minimum starting duty. Set a minimum duty of 20000 for the fan to spin at low temperatures. |
| DHT22 read error | Data line issue | Ensure a 10 kΩ pull-up resistor is on the DATA line if using long cable runs. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [61 - Pico DC Motor L298N](61-pico-dc-motor-l298n.md)
- [106 - Pico DHT22 Temperature Humidity LCD](106-pico-dht22-temperature-humidity-lcd.md)
- [99 - Pico DC Motor L298N Potentiometer LCD](99-pico-dc-motor-l298n-potentiometer-lcd.md)
