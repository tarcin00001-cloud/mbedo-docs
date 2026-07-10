# 117 - Pico Servo Radar Scanner OLED

Build a sweeping sonar radar console that rotates an HC-SR04 sensor on a servo motor through 180 degrees, measures distance at each angle, and plots a sweep map on an SSD1306 OLED display.

## Goal
Learn how to sweep a servo through angular positions, trigger ultrasonic measurements at each step, and plot a graphical radar sweep on an SSD1306 OLED screen in MicroPython.

## What You Will Build
A sonar radar scanner:
- **Servo Motor (GP10)**: Sweeps the HC-SR04 sensor from 0° to 180° and back.
- **HC-SR04 Ultrasonic Sensor (GP16 TRIG, GP17 ECHO)**: Measures distance at each sweep angle.
- **SSD1306 OLED (GP4, GP5)**: Plots the radar sweep as a graphical line and displays the current angle and distance.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes (standard SG90) |
| HC-SR04 Ultrasonic Sensor | `hcsr04` | Yes | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Servo Motor | Control (PWM) | GP10 | Orange | PWM servo signal |
| Servo Motor | Power (+) | 5V (VBUS) | Red | Servo power supply (5V) |
| HC-SR04 | VCC | 5V (VBUS) | Red | Sensor power (requires 5V) |
| HC-SR04 | GND | GND | Black | Ground reference |
| HC-SR04 | TRIG | GP16 | Orange | Trigger pulse output |
| HC-SR04 | ECHO | GP17 | Yellow | Echo pulse input |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the servo control pin to GP10 and power to 5V VBUS. Connect HC-SR04 TRIG to GP16, ECHO to GP17 (with a 1kΩ/2kΩ voltage divider for 3.3V). Connect the OLED to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, PWM, I2C
import utime
import machine
import math
import ssd1306

# Set up Servo on GP10 (50 Hz PWM)
servo_pwm = PWM(Pin(10))
servo_pwm.freq(50)

trig = Pin(16, Pin.OUT)
echo = Pin(17, Pin.IN)

# Initialize shared I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# Servo duty cycle limits
MIN_DUTY = 1638 # 0 degrees
MAX_DUTY = 8192 # 180 degrees

# Radar display center and scale
CX, CY   = 64, 55  # Radar origin (bottom centre)
MAX_DIST = 80       # Max visible distance in cm

def set_angle(angle):
    duty = MIN_DUTY + int(angle * (MAX_DUTY - MIN_DUTY) / 180)
    servo_pwm.duty_u16(duty)

def measure_distance():
    trig.value(0)
    utime.sleep_us(2)
    trig.value(1)
    utime.sleep_us(10)
    trig.value(0)
    duration = machine.time_pulse_us(echo, 1, 30000)
    if duration < 0:
        return MAX_DIST
    return min(MAX_DIST, duration / 58.0)

def polar_to_xy(angle_deg, dist_cm):
    """Convert polar coordinates to OLED pixel coordinates."""
    angle_rad = math.radians(180 - angle_deg) # Flip for display orientation
    scale = 50 / MAX_DIST # Scale to fit display height
    px = int(CX + dist_cm * scale * math.cos(angle_rad))
    py = int(CY - dist_cm * scale * math.sin(angle_rad))
    return px, py

# Sweep angles
STEP = 10  # Degrees per step

oled.fill(0)
oled.text("Radar Scanner", 0, 0)
oled.text("Initializing...", 0, 20)
oled.show()
utime.sleep(1)

print("Servo radar scanner active.")

while True:
    oled.fill(0) # Clear screen
    # Draw baseline arc
    oled.hline(14, 55, 100, 1)
    
    # Forward sweep: 0 to 180 degrees
    for angle in range(0, 181, STEP):
        set_angle(angle)
        utime.sleep_ms(80) # Allow servo to settle
        dist = measure_distance()
        
        # Draw radar line
        px, py = polar_to_xy(angle, dist)
        oled.line(CX, CY, px, py, 1)
        
        # Draw blip at detected distance
        oled.pixel(px, py, 1)
        
        print("Angle: {:3d} | Dist: {:.1f} cm".format(angle, dist))
        
    # Display angle and last reading in corner
    oled.text("{}d {:.0f}cm".format(angle, dist), 0, 0)
    oled.show()
    
    # Reverse sweep: 180 back to 0
    for angle in range(180, -1, -STEP):
        set_angle(angle)
        utime.sleep_ms(80)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Servo Motor**, **HC-SR04 Sensor**, and **SSD1306 OLED** onto the canvas.
2. Connect Servo to **GP10**, HC-SR04 TRIG to **GP16** and ECHO to **GP17**, and OLED to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Observe the servo sweeping and the OLED plotting the radar sweep diagram.

## Expected Output
```
Servo radar scanner active.
Angle:   0 | Dist: 45.2 cm
Angle:  10 | Dist: 80.0 cm
Angle:  90 | Dist: 32.1 cm
```
(On OLED: graphical radar lines radiating from the bottom centre at each measured angle.)

## Expected Canvas Behavior
- The servo motor component on the canvas sweeps and the OLED displays a growing radar-style sweep map with lines indicating measured distances at each angle.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `math.radians(180 - angle_deg)` | Converts the servo angle to a display angle, flipping orientation so 0° points left and 180° points right. |
| `oled.line(CX, CY, px, py, 1)` | Draws a radar sweep line from the origin to the detected obstacle position. |

## Hardware & Safety Concept: Mechanical Turret Assemblies
Real sonar turrets physically mount the transducer on the servo horn using 3D-printed brackets. The sensor must be centred on the servo's axis of rotation to prevent parallax errors. Counterweights are added to balance the mass and reduce servo motor strain during oscillation.

## Try This! (Challenges)
1. **Detection Alert**: Add a buzzer on GP13 that sounds when a detected distance is below 30 cm.
2. **Smoother Sweep**: Reduce `STEP` to 5 degrees for higher angular resolution, and increase the servo settle time accordingly.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo jitters at each step | Electrical noise from servo | Add a 100µF capacitor across the servo power rails. |
| All distances read MAX_DIST | ECHO pin voltage too high | Add a 1kΩ/2kΩ voltage divider on the ECHO line. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [59 - Pico Servo Knob](59-pico-servo-knob.md)
- [66 - Pico Ultrasonic Radar Servo](66-pico-ultrasonic-radar-servo.md)
- [109 - Pico Ultrasonic Distance OLED](109-pico-ultrasonic-distance-oled.md)
