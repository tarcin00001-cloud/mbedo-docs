# 133 - Pico Autonomous Guard HC-SR04 PIR Buzzer LCD

Build an autonomous perimeter guard that combines HC-SR04 distance measurement with PIR motion detection, evaluates a compound threat condition, activates multi-output alerts, and displays threat level on an I2C LCD.

## Goal
Learn how to fuse data from two independent sensors (ultrasonic distance and PIR motion) using compound Boolean logic to classify threat levels, control multiple outputs (buzzer, LED, relay), and display the threat assessment on an I2C character LCD in MicroPython.

## What You Will Build
An autonomous perimeter guard:
- **HC-SR04 Ultrasonic Sensor (GP16, GP17)**: Detects object distance. Close proximity = elevated threat.
- **PIR Motion Sensor (GP14)**: Detects movement in the monitored zone.
- **Active Buzzer (GP15)**: Siren pattern scales with threat level.
- **LED (GP13)**: Flashes at threat-level rate.
- **Relay (GP10)**: Activates external siren/lights at HIGH threat.
- **I2C 16x2 LCD (GP4, GP5)**: Displays distance, motion state, and threat level.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hcsr04` | Yes | Yes |
| PIR Motion Sensor | `button` | Yes (digital output) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 | VCC | 5V (VBUS) | Red | Sensor power |
| HC-SR04 | GND | GND | Black | Ground reference |
| HC-SR04 | TRIG | GP16 | Orange | Trigger pulse output |
| HC-SR04 | ECHO | GP17 | Yellow | Echo input (1kΩ/2kΩ divider required) |
| PIR Sensor | VCC | 5V (VBUS) | Red | PIR power |
| PIR Sensor | GND | GND | Black | Ground reference |
| PIR Sensor | OUT | GP14 | Blue | Motion output (HIGH = motion) |
| Active Buzzer | VCC (+) | GP15 | Red | Alert siren output |
| LED | Anode (+, via 330 Ω) | GP13 | Orange | Threat indicator |
| Relay Module | IN | GP10 | Purple | External siren/light relay |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Add a 1kΩ/2kΩ voltage divider on the HC-SR04 ECHO line to bring 5V down to 3.3V for the Pico GPIO. Connect PIR to GP14, buzzer to GP15, LED to GP13, relay to GP10, and LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime, machine
from machine_lcd import I2cLcd

trig      = Pin(16, Pin.OUT)
echo      = Pin(17, Pin.IN)
pir       = Pin(14, Pin.IN, Pin.PULL_DOWN)
buzzer    = Pin(15, Pin.OUT)
alert_led = Pin(13, Pin.OUT)
relay     = Pin(10, Pin.OUT)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Threat distance thresholds (cm)
DIST_HIGH   = 80   # HIGH threat if object within 80 cm
DIST_MEDIUM = 200  # MEDIUM if within 200 cm

relay.value(0); buzzer.value(0); alert_led.value(0)

def measure_dist():
    trig.value(0); utime.sleep_us(2)
    trig.value(1); utime.sleep_us(10)
    trig.value(0)
    d = machine.time_pulse_us(echo, 1, 30000)
    return 999.0 if d < 0 else d / 58.0

def classify_threat(dist, motion):
    """Return (level string, relay, buzz_ms, pause_ms)."""
    if motion and dist < DIST_HIGH:
        return "HIGH    ", True, 150, 100
    elif motion or dist < DIST_HIGH:
        return "MEDIUM  ", False, 200, 400
    elif dist < DIST_MEDIUM:
        return "LOW     ", False, 100, 900
    else:
        return "CLEAR   ", False, 0, 500

lcd.clear(); lcd.putstr("Perimeter Guard")
utime.sleep(2)
print("Autonomous guard active.")

while True:
    dist   = measure_dist()
    motion = (pir.value() == 1)

    level, relay_on, buzz_on_ms, buzz_off_ms = classify_threat(dist, motion)

    relay.value(1 if relay_on else 0)

    # LCD update
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("D:{:.0f}cm M:{} ".format(dist, "Y" if motion else "N"))
    lcd.move_to(0, 1)
    lcd.putstr("Threat: " + level)

    print("Dist:{:.0f}cm Motion:{} Threat:{}".format(dist, motion, level.strip()))

    # Alert pattern
    if buzz_on_ms > 0:
        alert_led.value(1)
        buzzer.value(1)
        utime.sleep_ms(buzz_on_ms)
        alert_led.value(0)
        buzzer.value(0)
        utime.sleep_ms(buzz_off_ms)
    else:
        alert_led.value(0)
        utime.sleep_ms(buzz_off_ms)
```

## What to Click in MbedO
1. Drag **Pico**, **HC-SR04**, **PIR Button**, **Active Buzzer**, **LED**, **Relay**, and **I2C LCD** onto the canvas.
2. Connect all components per the wiring table.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Decrease the HC-SR04 slider below 80 cm AND click the PIR button to trigger HIGH threat. Observe the relay close and buzzer alarm rate increase.

## Expected Output
```
Autonomous guard active.
Dist:999cm Motion:False Threat:CLEAR
Dist:150cm Motion:False Threat:LOW
Dist:60cm Motion:True Threat:HIGH
```

## Expected Canvas Behavior
- In CLEAR state: outputs are off. In LOW: slow buzzer pulses. In MEDIUM: faster pulses. In HIGH: relay closes, rapid buzzer pulses, LED flashes rapidly.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `motion and dist < DIST_HIGH` | AND gate for HIGH threat: requires both close proximity AND confirmed motion. |
| `classify_threat()` returns timing tuples | Separates threat classification from output control — the returned timing values drive the output pattern. |

## Hardware & Safety Concept: Sensor Fusion for Threat Classification
Using a single sensor for security creates false positives: a PIR alone may trigger on HVAC airflow; an ultrasonic alone may trigger on a stationary object. Requiring agreement between two independent sensing modalities (distance AND motion) dramatically reduces false-alarm rates — the same principle used in industrial intrusion detection systems.

## Try This! (Challenges)
1. **Graduated Relay**: Replace the digital relay with a PWM buzzer that increases frequency proportionally to threat score.
2. **SMS Alert**: Integrate an HC-05 Bluetooth connection to send a threat notification to a smartphone app.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Always HIGH threat | HC-SR04 reads short distance | Verify the ECHO voltage divider is present. Ensure clear line of sight for the sensor. |
| PIR not detecting motion | PIR in single-trigger mode | Move the PIR jumper to H (re-trigger) mode for sustained detection. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [120 - Pico Multi-Sensor Alarm System LCD](../intermediate/120-pico-multi-sensor-alarm-system-lcd.md)
- [115 - Pico Parking Sensor HC-SR04 Buzzer LCD](../intermediate/115-pico-parking-sensor-hcsr04-buzzer-lcd.md)
