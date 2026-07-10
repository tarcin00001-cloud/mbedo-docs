# 86 - Pico IR Obstacle Avoidance LCD

Build a proximity alert dashboard that displays obstacle alert warnings on an I2C LCD, flashes a warning LED, and sounds a warning buzzer when an IR obstacle sensor triggers.

## Goal
Learn how to read digital inputs from IR obstacle sensors, handle active-low logic levels, output status text to an I2C character LCD, and actuate visual/audio warnings in MicroPython.

## What You Will Build
A collision warning dashboard:
- **IR Proximity Sensor (GP14)**: Detects nearby surfaces (active LOW digital signal).
- **LED (GP15)**: Turns ON to provide a visual warning when an obstacle is detected.
- **Active Buzzer (GP13)**: Sounds a warning beep in sync with the LED.
- **I2C 16x2 LCD (GP4, GP5)**: Displays active status ("OBSTACLE!" or "PATH CLEAR") and warnings.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| IR Obstacle Sensor | `button` | Yes (represented by push button component) | Yes |
| LED (any colour) | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Sensor | VCC | 3.3V (3V3) or 5V | Red | Power line |
| IR Sensor | GND | GND | Black | Ground reference |
| IR Sensor | OUT | GP14 | Blue | Digital proximity signal (LOW on obstacle) |
| LED | Anode (+, longer leg) | GP15 | Orange | Warning indicator output pin |
| LED | Cathode (−, shorter leg) | GND | Black | Shared return path to ground |
| 330 Ω Resistor | Either leg | In series with LED Anode | — | Limits LED current |
| Active Buzzer | VCC (+) | GP13 | Red | Warning sound output pin |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the IR sensor output to GP14, LED anode to GP15 through the resistor, and active buzzer VCC to GP13. Connect the LCD to GP4/GP5. All ground pins are connected together.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

# IR proximity sensors typically output LOW (0V) when an obstacle is detected
ir_sensor = Pin(14, Pin.IN, Pin.PULL_UP)
warning_led = Pin(15, Pin.OUT)
warning_buz = Pin(13, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Start silent and dark
warning_led.value(0)
warning_buz.value(0)

lcd.clear()
lcd.putstr("Proximity Panel")
lcd.move_to(0, 1)
lcd.putstr("Console Active")
utime.sleep(1)

# Initial screen update
lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("PATH CLEAR")
lcd.move_to(0, 1)
lcd.putstr("Status: SAFE")

prev_state = 1
ir_sensor.value()

print("Proximity warning system active.")

while True:
    curr_state = ir_sensor.value()
    obstacle_detected = (curr_state == 0) # Active-LOW on obstacle
    
    # Check for transition change
    if curr_state != prev_state:
        lcd.clear()
        if obstacle_detected:
            lcd.move_to(0, 0)
            lcd.putstr("! OBSTACLE DET !")
            lcd.move_to(0, 1)
            lcd.putstr("COLLISION RISK")
            print(">> COLLISION WARNING: Obstacle detected!")
        else:
            lcd.move_to(0, 0)
            lcd.putstr("PATH CLEAR")
            lcd.move_to(0, 1)
            lcd.putstr("Status: SAFE")
            print(">> Path clear.")
        prev_state = curr_state
        
    if obstacle_detected:
        warning_led.value(1) # Turn warning LED ON
        warning_buz.value(1) # Turn warning buzzer ON
        utime.sleep_ms(150)   # Active alarm pulse
        warning_led.value(0)
        warning_buz.value(0)
        utime.sleep_ms(100)
    else:
        warning_led.value(0)
        warning_buz.value(0)
        utime.sleep_ms(50) # Normal polling speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button** (representing IR sensor), **LED**, **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect IR Button to **GP14**, LED to **GP15**, Buzzer to **GP13**, and LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click and hold the button component on the canvas to simulate an obstacle and observe the LED, buzzer, and LCD display update.

## Expected Output
```
Proximity warning system active.
>> COLLISION WARNING: Obstacle detected!
```
(On screen: "PATH CLEAR / Status: SAFE" changing to "! OBSTACLE DET ! / COLLISION RISK" when triggered.)

## Expected Canvas Behavior
- Clicking the button component causes the LED and buzzer components on the canvas to pulse ON and OFF rapidly, and the LCD screen to update with warning text.
- Releasing the button silences the alarm, turns the LED OFF, and returns the LCD to the safe message.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `ir_sensor.value() == 0` | Detects when the IR sensor's output is pulled LOW, indicating a nearby object is reflecting IR light. |
| `lcd.putstr("! OBSTACLE DET !")` | Renders a high-priority warning message on the LCD display buffer. |

## Hardware & Safety Concept: Optical Proximity Ranges
Infrared obstacle sensors use an infrared transmitter and receiver pair. Proximity detection range (typically **2 to 30 cm**) is set by adjusting the onboard sensitivity potentiometer. Because ambient light (especially sunlight) contains infrared waves, these sensors can false-trigger outdoors. They are best suited for indoor applications like robot obstacle avoidance.

## Try This! (Challenges)
1. **Latching Hazard Mode**: Modify the code to latch the alarm state ON once triggered, requiring a reset button on GP12 to clear it.
2. **Reverse Alert**: sound the buzzer faster (increase beep frequency) as the object gets closer (simulating parking assist sensors).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Sensitivity set too high | Rotate the sensitivity screw counterclockwise on the IR sensor board to decrease the range. |
| Sensor does not detect objects | Direct sunlight interference | Move the sensor away from direct sunlight or bright incandescent lights. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [75 - Pico IR Obstacle Avoidance LED](75-pico-ir-obstacle-avoidance-led.md)
- [85 - Pico Motion Security PIR LCD](85-pico-motion-security-pir-lcd.md)
