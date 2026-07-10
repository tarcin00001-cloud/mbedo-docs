# 77 - Pico Potentiometer Servo LCD

Build a rotary steering angle console that reads a potentiometer position, steers a servo motor, and displays the active angle coordinates on an I2C character LCD.

## Goal
Learn how to read an analog potentiometer value (0–65535), map it to servo duty cycles (0–180 degrees), and format/print the values on an I2C character LCD in MicroPython.

## What You Will Build
A rotary steering dashboard:
- **Potentiometer (GP26)**: Reads steering adjustments.
- **Servo Motor (GP10)**: Steers to match the potentiometer knob position.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the current angle (0 to 180 degrees) and a visual progress bar.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Left Pin (Pin 1) | GND | Black | Ground reference |
| Potentiometer | Wiper (Pin 2 - centre) | GP26 | Yellow | Analog input pin |
| Potentiometer | Right Pin (Pin 3) | 3.3V (3V3) | Red | High voltage reference |
| Servo Motor | Control (PWM) | GP10 | Orange | PWM control signal |
| Servo Motor | Power (+) | 5V (VBUS) | Red | Servo power supply (5V) |
| Servo Motor | GND (−) | GND | Black | Shared return path to ground |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect both the servo and LCD power lines. The LCD and servo control pins connect to GP4/GP5 and GP10 respectively. The potentiometer wiper pin connects to GP26.

## Code
```python
from machine import Pin, ADC, PWM, I2C
import utime
from machine_lcd import I2cLcd

pot = ADC(26) # GP26 = ADC channel 0

# Set up Servo on GP10 (50 Hz PWM)
servo = PWM(Pin(10))
servo.freq(50)

# Set up I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Servo duty cycle bounds
MIN_DUTY = 1638
MAX_DUTY = 8192

lcd.clear()
lcd.putstr("Steering Node")
lcd.move_to(0, 1)
lcd.putstr("Initializing...")
utime.sleep(1)

while True:
    raw = pot.read_u16()
    
    # Map raw value (0 - 65535) to duty range (MIN_DUTY - MAX_DUTY)
    duty = MIN_DUTY + int(raw * (MAX_DUTY - MIN_DUTY) / 65535)
    servo.duty_u16(duty)
    
    # Map raw value to angle (0 to 180 degrees)
    angle = int(raw * 180 / 65535)
    
    # Calculate visual bar length for the LCD (0 to 16 characters)
    bar_chars = int(raw * 16 / 65535)
    bar_str = "=" * bar_chars + " " * (16 - bar_chars)
    
    # Update LCD Display
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("Angle: " + str(angle) + " deg")
    lcd.move_to(0, 1)
    lcd.putstr(bar_str)
    
    print("Angle:", angle, "deg | Duty:", duty)
    utime.sleep_ms(150) # Smooth update speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, **Servo Motor**, and **I2C LCD** onto the canvas.
2. Connect Potentiometer wiper to **GP26**. Connect Servo Control to **GP10** and LCD to **GP4/GP5**.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the potentiometer slider on the canvas and observe the servo and LCD display update.

## Expected Output
```
Angle: 0 deg | Duty: 1638
Angle: 90 deg | Duty: 4915
Angle: 180 deg | Duty: 8192
```
(On screen: "Angle: X deg" on line 1, and a progress bar made of `===` characters on line 2.)

## Expected Canvas Behavior
- The servo motor shaft rotates and the LCD text/progress bar updates in real-time response to the position of the potentiometer slider.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `bar_chars = int(raw * 16 / 65535)` | Scales the 16-bit ADC value to fit within the 16-character width of the LCD display. |
| `"=" * bar_chars + " "` | Creates a text-based progress bar by repeating the `=` character. |

## Hardware & Safety Concept: Decoupling Capacitors
When servo motors start rotating, they draw high current, which can cause electrical noise on the power rails. This noise can interfere with sensitive I2C displays, causing them to show random blocks or freeze. Adding a **100 µF electrolytic capacitor** across the power and ground rails near the servo acts as a reservoir, filtering out voltage drops and protecting other components.

## Try This! (Challenges)
1. **Flashing Lock Indicator**: Connect an LED on GP13 and flash it if the steering angle reaches either limits (0 or 180 degrees).
2. **Reverse Steering Switch**: Connect a button on GP12 to reverse the steering direction.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD displays random blocks or freezes | Electrical noise from servo | Add a decoupling capacitor across the power rails, or power the servo from a separate supply. |
| Servo does not rotate | Power rail issue | Connect the servo's power (+) pin to the 5V VBUS pin, not 3.3V. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [11 - Pico Potentiometer Serial](11-pico-potentiometer-serial.md)
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [59 - Pico Servo Knob](59-pico-servo-knob.md)
