# 66 - Pico Ultrasonic Radar Servo

Build a scanning sonar radar that sweeps an ultrasonic distance sensor using a servo motor and logs angle/distance data to the serial monitor.

## Goal
Learn how to sweep a servo motor continuously, trigger rangefinder readings at specific angles, and stream structured coordinate datasets (Angle, Distance) in MicroPython.

## What You Will Build
A scanning sonar radar:
- **Servo Motor (GP10)**: Sweeps back and forth from 15 to 165 degrees in 5-degree steps.
- **HC-SR04 Sensor (Trig GP16, Echo GP17)**: Mounted on the servo horn to measure distance to targets at each step.
- **Serial Output**: Streams coordinates (e.g. `Angle,Distance`) to the terminal for plotting.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Servo Motor | Control (PWM) | GP10 | Orange | PWM angle signal |
| Servo Motor | Power (+) | 5V (VBUS) | Red | Servo power supply (5V) |
| Servo Motor | GND (−) | GND | Black | Shared return path to ground |
| HC-SR04 | VCC | 5V (VBUS) | Red | Sensor requires 5V power |
| HC-SR04 | Trig | GP16 | Yellow | Digital output pulse |
| HC-SR04 | Echo | GP17 | Blue | Digital input (requires 5V to 3.3V voltage divider) |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect both the servo and HC-SR04 power pins to the 5V VBUS pin. Connect the servo control pin to GP10, HC-SR04 Trig to GP16, and Echo to GP17 through a 5V-to-3.3V voltage divider.

## Code
```python
from machine import Pin, PWM, ADC
import utime

# Set up Servo on GP10 (50 Hz PWM)
servo = PWM(Pin(10))
servo.freq(50)

# Set up HC-SR04 on GP16/GP17
trigger = Pin(16, Pin.OUT)
echo    = Pin(17, Pin.IN)

MIN_DUTY = 1638 # ~0.5ms (0 degrees)
MAX_DUTY = 8192 # ~2.5ms (180 degrees)

def set_servo_angle(angle):
    duty = MIN_DUTY + int(angle * (MAX_DUTY - MIN_DUTY) / 180)
    servo.duty_u16(duty)

def get_distance():
    # Trigger sensor: send 10-microsecond pulse
    trigger.value(0)
    utime.sleep_us(5)
    trigger.value(1)
    utime.sleep_us(10)
    trigger.value(0)
    
    # Measure pulse width
    duration = machine.time_pulse_us(echo, 1, 30000)
    if duration > 0:
        return (duration * 0.0343) / 2
    return -1

print("Radar Sweep Initialized.")
print("Format: Angle_deg,Distance_cm")

while True:
    # Sweep from 15 to 165 degrees (forward)
    for angle in range(15, 166, 5):
        set_servo_angle(angle)
        utime.sleep_ms(150) # Allow servo to reach position
        
        dist = get_distance()
        print(angle, ",", round(dist, 1))
        
    # Sweep from 165 to 15 degrees (reverse)
    for angle in range(165, 14, -5):
        set_servo_angle(angle)
        utime.sleep_ms(150) # Allow servo to reach position
        
        dist = get_distance()
        print(angle, ",", round(dist, 1))
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Servo Motor**, and **HC-SR04 Sensor** onto the canvas.
2. Connect Servo control to **GP10**, HC-SR04 Trig to **GP16**, and Echo to **GP17**. Connect power and grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the servo sweeps and rangefinder distance measurements in the terminal.

## Expected Output
```
Radar Sweep Initialized.
Format: Angle_deg,Distance_cm
15 , 120.4
20 , 118.2
25 , 45.1
```

## Expected Canvas Behavior
- The servo motor shaft component sweeps back and forth on the canvas.
- The terminal prints updated angles and distances at each step.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `for angle in range(15, 166, 5)` | Sweeps the search angle from 15 to 165 degrees in 5-degree steps. |
| `utime.sleep_ms(150)` | Delay that gives the servo physical time to rotate to the new step position before taking a reading. |

## Hardware & Safety Concept: Echo Settling Delays
When firing ultrasonic sound pulses rapidly in a scanning sweep, old echo pulses can bounce off distant walls and return during the next measurement window, causing noise. To prevent cross-talk noise, radar controllers include a short settling delay (typically **50 to 100 ms**) between readings to let old sound waves dissipate.

## Try This! (Challenges)
1. **Target Alert LED**: Connect an LED on GP14 and turn it ON if an obstacle is detected within 30 cm at any sweep angle.
2. **Visual LED Tracker**: Connect three LEDs and light them up based on target direction (left, center, right).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Radar readings are all -1 | Echo divider missing | Ensure the 1 kΩ and 2 kΩ resistors are wired correctly to GP17 to scale the echo signal down to 3.3V. |
| Servo does not sweep | Power supply overload | Connect the servo's power (+) pin to the 5V VBUS pin, or use an external 5V power supply. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [54 - Pico LCD Distance HC-SR04](54-pico-lcd-distance-hc-sr04.md)
- [58 - Pico OLED Distance HC-SR04](58-pico-oled-distance-hc-sr04.md)
- [59 - Pico Servo Knob](59-pico-servo-knob.md)
