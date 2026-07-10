# 54 - Pico LCD Distance HC-SR04

Build a non-contact digital rangefinder that measures obstacle distance using an HC-SR04 ultrasonic sensor and displays the result on an I2C character LCD.

## Goal
Learn how to trigger ultrasonic sound pulses, measure microsecond response echo times, calculate distances in centimeters, and update an I2C 16x2 LCD in MicroPython.

## What You Will Build
A digital distance meter:
- **HC-SR04 Sensor (Trig GP16, Echo GP17)**: Measures distance to target.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the calculated distance in centimeters and shows a simple visual text scale (e.g. "NEAR" or "SAFE").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 | VCC | 5V (VBUS) | Red | Sensor requires 5V power |
| HC-SR04 | Trig (Trigger) | GP16 | Yellow | Digital output pulse |
| HC-SR04 | Echo | GP17 | Blue | Digital input (requires 5V to 3.3V voltage divider) |
| HC-SR04 | GND | GND | Black | Ground reference |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The HC-SR04 requires 5V to function. Because its Echo pin outputs a 5V signal, you must use a **voltage divider** (e.g. a 1 kΩ and 2 kΩ resistor in series to GND) to safely step the signal down to 3.3V before connecting it to GP17.
>
> **Echo Divider Wiring:** Echo pin → 1 kΩ resistor → GP17 → 2 kΩ resistor → GND.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

trigger = Pin(16, Pin.OUT)
echo    = Pin(17, Pin.IN)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

lcd.clear()
lcd.putstr("Rangefinder Node")
lcd.move_to(0, 1)
lcd.putstr("Initializing...")
utime.sleep(1)

while True:
    # 1. Trigger sensor: send 10-microsecond pulse
    trigger.value(0)
    utime.sleep_us(5)
    trigger.value(1)
    utime.sleep_us(10)
    trigger.value(0)
    
    # 2. Measure pulse width (time the echo pin stays HIGH)
    # timeout of 30ms maps to max range (~5 meters)
    duration = machine.time_pulse_us(echo, 1, 30000)
    
    if duration > 0:
        # Distance = (time * speed of sound) / 2
        # Speed of sound is 343 m/s = 0.0343 cm/us
        distance = (duration * 0.0343) / 2
        
        # Determine status label
        status = "SAFE"
        if distance < 15.0:
            status = "CRITICAL"
        elif distance < 40.0:
            status = "WARN"
            
        # 3. Update LCD Display
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Dist: " + str(round(distance, 1)) + " cm")
        
        lcd.move_to(0, 1)
        lcd.putstr("Status: " + status)
        
        print("Distance:", round(distance, 1), "cm | Status:", status)
    else:
        lcd.clear()
        lcd.putstr("Out of Range")
        print("Pulse timeout / out of range")
        
    utime.sleep_ms(300) # Update rate
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-SR04 Sensor**, and **I2C LCD** onto the canvas.
2. Connect Trig to **GP16**, Echo to **GP17**, and LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the rangefinder value on the canvas and observe the LCD updating.

## Expected Output
```
Distance: 120.4 cm | Status: SAFE
Distance: 32.1 cm | Status: WARN
Distance: 12.5 cm | Status: CRITICAL
```
(On screen: "Dist: X cm" and "Status: SAFE/WARN/CRITICAL" on respective lines.)

## Expected Canvas Behavior
- The LCD component displays the distance value matching the position of the HC-SR04 slider.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `machine.time_pulse_us(echo, 1, 30000)` | Measures how long GP17 stays HIGH, with a timeout limit of 30,000 microseconds. |
| `(duration * 0.0343) / 2` | Converts the travel time to distance in centimeters based on the speed of sound. |

## Hardware & Safety Concept: Echo Pin Voltage Dividers
Connecting a 5V signal directly to a Pico's 3.3V GPIO pin can damage the pin over time. Using a voltage divider (resistor network) is a simple and reliable way to step 5V signals down to a safe 3.3V level. Always verify that your sensors' digital output voltages match the tolerance of your microcontroller.

## Try This! (Challenges)
1. **Auditory Warning Alert**: Connect a buzzer on GP14 and sound a rapid warning beep when the distance falls below 15 cm.
2. **Dynamic Backlight**: Turn OFF the LCD backlight automatically when the path is clear (distance > 100 cm) to save energy.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD reads "Out of Range" constantly | Echo voltage divider missing | Ensure the 1 kΩ and 2 kΩ resistors are wired correctly to GP17 to scale the echo signal down to 3.3V. |
| Distance readings are unstable | Reflections or obstacles | Place the sensor facing a flat, solid surface. Soft or angled surfaces can absorb or scatter the sound waves. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [58 - Pico OLED Distance HC-SR04](58-pico-oled-distance-hc-sr04.md)
