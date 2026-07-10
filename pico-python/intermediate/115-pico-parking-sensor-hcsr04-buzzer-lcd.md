# 115 - Pico Parking Sensor HC-SR04 Buzzer LCD

Build a car parking proximity assistant that measures distance using an HC-SR04 sensor, sounds an urgency beeping buzzer at progressively faster rates as an object approaches, and displays the distance on an I2C LCD.

## Goal
Learn how to implement variable-rate buzzer pulsing tied to measured ultrasonic distance values, and display proximity zone warnings on an I2C character LCD in MicroPython.

## What You Will Build
A parking proximity assistant:
- **HC-SR04 Ultrasonic Sensor (GP16 TRIG, GP17 ECHO)**: Measures distance to the nearest obstacle.
- **Passive Buzzer (GP15)**: Beeps faster as the object gets closer (slow → medium → rapid → continuous).
- **I2C 16x2 LCD (GP4, GP5)**: Displays distance and proximity zone ("SAFE", "CAUTION", "DANGER!", "STOP!").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hcsr04` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes (passive required) |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 | VCC | 5V (VBUS) | Red | Sensor power (requires 5V) |
| HC-SR04 | GND | GND | Black | Ground reference |
| HC-SR04 | TRIG | GP16 | Orange | Trigger pulse output |
| HC-SR04 | ECHO | GP17 | Yellow | Echo pulse input |
| Passive Buzzer | Signal (+) | GP15 | Grey | Warning beep signal (PWM) |
| Passive Buzzer | GND (−) | GND | Black | Shared return path to ground |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** On real hardware, add a 1kΩ/2kΩ voltage divider on the HC-SR04 ECHO line to reduce 5V to 3.3V for the Pico GPIO. Connect the passive buzzer signal to GP15. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, PWM, I2C
import utime
import machine
from machine_lcd import I2cLcd

trig   = Pin(16, Pin.OUT)
echo   = Pin(17, Pin.IN)
buzzer = PWM(Pin(15))
buzzer.duty_u16(0)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Proximity zone thresholds (cm)
ZONE_STOP   = 15
ZONE_DANGER = 30
ZONE_CAUTION = 60
ZONE_SAFE   = 100

def measure_distance():
    trig.value(0)
    utime.sleep_us(2)
    trig.value(1)
    utime.sleep_us(10)
    trig.value(0)
    duration = machine.time_pulse_us(echo, 1, 30000)
    if duration < 0:
        return -1.0
    return duration / 58.0

def beep(freq, duration_ms):
    buzzer.freq(freq)
    buzzer.duty_u16(32768)
    utime.sleep_ms(duration_ms)
    buzzer.duty_u16(0)

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Parking Sensor")
lcd.move_to(0, 1)
lcd.putstr("Ready...")
utime.sleep(1)

print("Parking sensor active.")

while True:
    dist_cm = measure_distance()
    
    if dist_cm < 0:
        zone = "CLEAR"
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("No object")
        lcd.move_to(0, 1)
        lcd.putstr("Zone: CLEAR")
        utime.sleep_ms(300)
    elif dist_cm <= ZONE_STOP:
        zone = "STOP!"
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Dist:{:.0f}cm !!".format(dist_cm))
        lcd.move_to(0, 1)
        lcd.putstr("Zone: STOP!")
        # Continuous tone
        buzzer.freq(880)
        buzzer.duty_u16(32768)
        utime.sleep_ms(200)
        buzzer.duty_u16(0)
        utime.sleep_ms(50)
    elif dist_cm <= ZONE_DANGER:
        zone = "DANGER!"
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Dist: {:.0f} cm".format(dist_cm))
        lcd.move_to(0, 1)
        lcd.putstr("Zone: DANGER!")
        beep(880, 80)
        utime.sleep_ms(120)
    elif dist_cm <= ZONE_CAUTION:
        zone = "CAUTION"
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Dist: {:.0f} cm".format(dist_cm))
        lcd.move_to(0, 1)
        lcd.putstr("Zone: CAUTION")
        beep(660, 80)
        utime.sleep_ms(320)
    else:
        zone = "SAFE"
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("Dist: {:.0f} cm".format(dist_cm))
        lcd.move_to(0, 1)
        lcd.putstr("Zone: SAFE")
        beep(440, 80)
        utime.sleep_ms(620)
        
    print("Dist: {:.1f} cm | Zone: {}".format(dist_cm if dist_cm >= 0 else 0, zone))
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-SR04 Sensor**, **Passive Buzzer**, and **I2C LCD** onto the canvas.
2. Connect HC-SR04 TRIG to **GP16** and ECHO to **GP17**. Connect Buzzer to **GP15** and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Decrease the HC-SR04 distance slider and observe the buzzer rate and LCD zone changing.

## Expected Output
```
Parking sensor active.
Dist: 80.0 cm | Zone: SAFE
Dist: 40.0 cm | Zone: CAUTION
Dist: 20.0 cm | Zone: DANGER!
Dist: 10.0 cm | Zone: STOP!
```

## Expected Canvas Behavior
- As the HC-SR04 slider distance decreases, the buzzer component beeps progressively faster and the LCD zone indicator changes through SAFE → CAUTION → DANGER! → STOP!

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `utime.sleep_ms(620)` | Slow beep interval in the SAFE zone. Shorter intervals in DANGER and STOP zones create urgency. |
| `buzzer.duty_u16(32768)` | Opens the PWM at 50% duty to produce a buzzer tone. |

## Hardware & Safety Concept: Ultrasonic Parking Sensors
Commercial parking sensors use arrays of ultrasonic transceivers to cover wide detection angles. Each sensor covers a 60-degree arc and the combined array eliminates blind spots. The beep rate is proportional to the distance: ISO 16840-2 defines the standard beep patterns for automotive parking aids.

## Try This! (Challenges)
1. **Graphical Bar**: Add a proportional fill bar on the LCD second line showing remaining distance graphically.
2. **Four-Zone Colour LEDs**: Connect Red/Yellow/Green LEDs and light the appropriate one for each proximity zone.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer pops without tone | Active buzzer connected | Use a **passive** buzzer — active buzzers have fixed oscillators and cannot produce variable tones. |
| Distance always -1 | ECHO voltage level too high | Add a 1kΩ/2kΩ voltage divider on the ECHO line to step 5V down to 3.3V for the Pico GPIO. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [54 - Pico LCD Distance HC-SR04](54-pico-lcd-distance-hc-sr04.md)
- [108 - Pico Ultrasonic Distance LCD](108-pico-ultrasonic-distance-lcd.md)
