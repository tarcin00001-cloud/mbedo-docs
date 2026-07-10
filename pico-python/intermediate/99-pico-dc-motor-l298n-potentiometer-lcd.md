# 99 - Pico DC Motor L298N Potentiometer LCD

Build a DC motor speed console that displays the current motor speed percentage and rotation direction on an I2C LCD screen.

## Goal
Learn how to read an analog potentiometer value (0–65535), calculate motor speed percentages, control H-bridge driver direction pins, and output status text to an I2C LCD in MicroPython.

## What You Will Build
A DC motor speed control panel:
- **Potentiometer (GP26)**: Middle position stops the motor. Turning left spins the motor backward (speed increases with rotation). Turning right spins the motor forward.
- **L298N Driver & DC Motor (GP10-12)**: Drives the DC motor.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the active direction ("FWD" or "REV") and speed percentage.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| L298N Driver Module | `l298n` | Yes | Yes |
| DC Motor | `dc_motor` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Left Pin (Pin 1) | GND | Black | Ground reference |
| Potentiometer | Wiper (Pin 2 - centre) | GP26 | Yellow | Analog input pin |
| Potentiometer | Right Pin (Pin 3) | 3.3V (3V3) | Red | High voltage reference |
| L298N Driver | ENA (Enable A) | GP10 | Orange | PWM speed control signal |
| L298N Driver | IN1 / IN2 | GP11 / GP12 | Yellow / Green | Motor direction lines |
| L298N Driver | VCC / GND | 5V (VBUS) / GND | Red / Black | Driver power supply |
| DC Motor | Motor Terminals | OUT1 / OUT2 | — | Driver motor outputs |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the L298N driver ENA pin to GP10 (PWM), IN1 to GP11, and IN2 to GP12. Connect the driver VCC to the Pico's 5V VBUS pin, and GND to GND. Connect the LCD to GP4/GP5. The potentiometer wiper pin connects to GP26.

## Code
```python
from machine import Pin, ADC, PWM, I2C
import utime
from machine_lcd import I2cLcd

pot = ADC(26) # GP26 = ADC channel 0

# Set up L298N control pins
ena = PWM(Pin(10)) # GP10 = PWM speed control
ena.freq(1000)      # Set PWM frequency to 1 kHz

in1 = Pin(11, Pin.OUT)
in2 = Pin(12, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Stop motor initially
in1.value(0)
in2.value(0)
ena.duty_u16(0)

lcd.clear()
lcd.putstr("Motor Console")
lcd.move_to(0, 1)
lcd.putstr("Initializing...")
utime.sleep(1)

while True:
    raw = pot.read_u16()
    
    # Check direction and speed relative to center point (32768)
    # Right of center (34000 to 65535) -> Forward rotation
    # Left of center (0 to 31000) -> Backward rotation
    # Center (31000 to 34000) -> Stop
    
    direction = "STOP"
    percent = 0
    
    if raw > 34000:
        # Scale speed: maps raw range to PWM duty (0 to 65535)
        diff = raw - 34000
        duty = int(diff * 65535 / (65535 - 34000))
        
        # Drive forward
        in1.value(1)
        in2.value(0)
        ena.duty_u16(duty)
        
        direction = "FORWARD"
        percent = duty * 100 // 65535
    elif raw < 31000:
        # Scale speed: maps raw range to PWM duty (0 to 65535)
        diff = 31000 - raw
        duty = int(diff * 65535 / 31000)
        
        # Drive backward
        in1.value(0)
        in2.value(1)
        ena.duty_u16(duty)
        
        direction = "REVERSE"
        percent = duty * 100 // 65535
    else:
        # Stop motor
        in1.value(0)
        in2.value(0)
        ena.duty_u16(0)
        
    # Update LCD Display
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("Dir: " + direction)
    lcd.move_to(0, 1)
    lcd.putstr("Speed: " + str(percent) + " %")
    
    print("Dir:", direction, "| Speed:", percent, "%")
    utime.sleep_ms(250) # Status update rate
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, **L298N Driver**, **DC Motor**, and **I2C LCD** onto the canvas.
2. Connect Potentiometer to **GP26**. Connect driver ENA to **GP10**, IN1 to **GP11**, IN2 to **GP12**, and LCD to **GP4/GP5**. Connect motor to driver outputs.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the potentiometer slider left and right to control the DC motor and check the LCD.

## Expected Output
```
Dir: STOP | Speed: 0 %
Dir: FORWARD | Speed: 45 %
Dir: REVERSE | Speed: 62 %
```
(On screen: "Dir: FORWARD" and "Speed: 45 %" on respective lines, updating.)

## Expected Canvas Behavior
- The DC motor component on the canvas rotates and the LCD screen updates to show the motor direction and speed percentage when the potentiometer slider is moved.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `ena.duty_u16(duty)` | Adjusts the motor speed using a PWM signal (longer duty cycle -> higher average voltage -> faster speed). |
| `in1.value(1); in2.value(0)` | Configures the H-bridge direction control pins to drive the motor forward. |
| `lcd.putstr("Dir: " + ...)` | Prints the motor direction on the LCD display. |

## Hardware & Safety Concept: H-Bridge Motor Control
DC motors require high current and can spin in both directions. An **H-bridge circuit** (like the L298N chip) uses four transistors to control current flow through the motor. By opening and closing diagonal pairs of transistors, the H-bridge can reverse the direction of current flow to spin the motor forward or backward.

## Try This! (Challenges)
1. **Dynamic Brake**: Implement active braking by setting IN1 and IN2 HIGH at the same time when stopping.
2. **Acceleration Ramping**: Smooth out speed transitions by adding code that ramps motor speed up or down gradually.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor hums but does not spin | PWM duty cycle too low | DC motors need a minimum voltage to start spinning. Ensure your speed control starts above 30% duty. |
| Motor spins in wrong direction | Motor leads reversed | Swap the two motor wire connections on the driver's output terminal block. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [61 - Pico DC Motor L298N](61-pico-dc-motor-l298n.md)
- [91 - Pico DC Motor L298N Potentiometer](91-pico-dc-motor-l298n-potentiometer.md)
