# 163 - Pico Car Parking Counter

Build a smart parking garage manager that monitors entry and exit spaces using rangefinders, controls access servo gates, and displays a slot counter on an LCD.

## Goal
Learn how to manage multiple servo motors and high-speed pulse sensors (HC-SR04) in parallel, implement entry/exit counter rules, and update LCD readouts using MicroPython.

## What You Will Build
An automated parking gate manager:
- **Entry Ultrasonic Sensor (Trig GP14, Echo GP15)**: Detects arriving cars.
- **Exit Ultrasonic Sensor (Trig GP16, Echo GP17)**: Detects departing cars.
- **Entry Gate Servo (GP10)**: Opens when a car arrives and slots are available.
- **Exit Gate Servo (GP11)**: Opens when a departing car approaches.
- **16x2 I2C LCD (GP4, GP5)**: Displays the active count of filled and empty slots.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensors | `ultrasonic` | Yes (two sensors) | Yes |
| Servo Motors | `servo` | Yes (two servos) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Entry Sensor | TRIG / ECHO | GP14 / GP15 | Yellow / Blue | Arriving car sensor |
| Entry Sensor | VCC / GND | 5V / GND | Red / Black | Power lines |
| Exit Sensor | TRIG / ECHO | GP16 / GP17 | Orange / White | Departing car sensor |
| Exit Sensor | VCC / GND | 5V / GND | Red / Black | Power lines |
| Entry Gate | Signal (PWM) | GP10 | Orange | Entry barrier servo |
| Exit Gate | Signal (PWM) | GP11 | Green | Exit barrier servo |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Both servos can be powered from the VBUS (5V) pin of the Pico. The ultrasonic sensors also require 5V. Ensure all grounds are connected together. The I2C LCD shares I2C Bus 0 (GP4/GP5).

## Code
```python
from machine import Pin, PWM, I2C
import utime, machine
from machine_lcd import I2cLcd

# Pin definitions
TRIG1 = Pin(14, Pin.OUT); ECHO1 = Pin(15, Pin.IN)  # Entry sensor
TRIG2 = Pin(16, Pin.OUT); ECHO2 = Pin(17, Pin.IN)  # Exit sensor

# Servos
srv1 = PWM(Pin(10)); srv1.freq(50)
srv2 = PWM(Pin(11)); srv2.freq(50)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

TOTAL_SLOTS = 5
occupied_slots = 0
DETECT_LIMIT = 15.0  # cm

def set_servo(pwm, angle):
    # Map 0-180 to duty_u16 (approx 1638 to 8192)
    duty = 1638 + int(angle * (8192 - 1638) / 180)
    pwm.duty_u16(duty)

def get_distance(trig, echo):
    trig.value(0)
    utime.sleep_us(2)
    trig.value(1)
    utime.sleep_us(10)
    trig.value(0)
    
    # 30ms timeout = ~5m max range
    duration = machine.time_pulse_us(echo, 1, 30000)
    if duration < 0:
        return 400.0
    return duration / 58.0

def update_display():
    lcd.clear()
    lcd.putstr("Parking Lot Node\n")
    lcd.putstr("Fill:{}/{} Free:{}".format(
        occupied_slots, TOTAL_SLOTS, TOTAL_SLOTS - occupied_slots))

# Initialise
set_servo(srv1, 0)
set_servo(srv2, 0)
update_display()
print("Smart parking gate counter engine online.")

while True:
    dist1 = get_distance(TRIG1, ECHO1)
    utime.sleep_ms(60)  # Settle delay
    dist2 = get_distance(TRIG2, ECHO2)
    
    # Process Entry Gate
    if 0 < dist1 < DETECT_LIMIT:
        if occupied_slots < TOTAL_SLOTS:
            lcd.clear()
            lcd.putstr("Welcome!\nGate Opening...")
            set_servo(srv1, 90)
            occupied_slots += 1
            utime.sleep(3)      # Hold open
            set_servo(srv1, 0)  # Close
            update_display()
        else:
            lcd.clear()
            lcd.putstr("Lot is FULL!\nWait for exit...")
            utime.sleep(2)
            update_display()
            
    # Process Exit Gate
    if 0 < dist2 < DETECT_LIMIT:
        if occupied_slots > 0:
            lcd.clear()
            lcd.putstr("Thank You!\nGate Opening...")
            set_servo(srv2, 90)
            occupied_slots -= 1
            utime.sleep(3)      # Hold open
            set_servo(srv2, 0)  # Close
            update_display()
            
    utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two HC-SR04 Sensors**, **two Servos**, and **I2C LCD** onto the canvas.
2. Connect HC-SR04 #1 to **GP14/GP15**, HC-SR04 #2 to **GP16/GP17**, Servos to **GP10/GP11**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the entry sensor slider below 15 cm to open the gate and increment the vehicle counter.

## Expected Output
Terminal:
```
Smart parking gate counter engine online.
```

## Expected Canvas Behavior
* Startup: LCD reads `Parking Lot Node` / `Fill:0/5 Free:5`. Gates are at 0°.
* Car arrives at entry sensor (dist1 = 10 cm): LCD reads `Welcome!`, Entry Servo rotates to 90° for 3 seconds, then returns to 0°. LCD updates to `Fill:1/5 Free:4`.
* Car arrives at exit sensor (dist2 = 10 cm): LCD reads `Thank You!`, Exit Servo rotates to 90° for 3 seconds, then returns to 0°. LCD updates to `Fill:0/5 Free:5`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `machine.time_pulse_us(echo, 1, 30000)` | Measures the high pulse duration of the ultrasonic echo pin with a timeout of 30 milliseconds. |
| `occupied_slots += 1` | Increments the vehicle counter when a car passes the entry gate scanner. |

## Hardware & Safety Concept: Entry/Exit Safety Interlocks
Automated parking gates use safety interlocks (typically ground induction loops or secondary photo-eye beams) directly under the gate. The gate servo will not close as long as these sensors detect a vehicle block, preventing the heavy barrier arm from dropping onto a car.

## Try This! (Challenges)
1. **Full Light Indicator**: Wire a Red LED on GP12 to light up when the lot is completely full, and a Green LED on GP13 when spaces are available.
2. **Audio Warning Beep**: Connect a buzzer on GP9 and sound a warning beep every time a gate opens.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Gate opens but doesn't close | Sensor bounce | Verify that you have a delay (e.g. `utime.sleep(3)`) after opening the gate to allow the car to fully pass before the gate closes. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [115 - Pico Parking Sensor HC-SR04 Buzzer LCD](../intermediate/115-pico-parking-sensor-hcsr04-buzzer-lcd.md)
- [151 - Pico Servo Pan-Tilt Camera Mount HC-SR04](151-pico-servo-pan-tilt-hcsr04-oled.md)
