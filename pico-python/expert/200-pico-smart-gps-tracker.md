# 200 - Pico Smart GPS Autonomous Tracker

Build an autonomous navigation computer that parses GPS NMEA sentences over UART, reads I2C compass headings, calculates distance to target waypoints, and steers a rudder servo.

## Goal
Learn how to parse raw serial GPS NMEA data streams, apply trigonometric calculations (Haversine formula approximations) to determine bearing and distance, and implement closed-loop steering controls in MicroPython.

## What You Will Build
An autonomous waypoint tracking computer:
- **GPS Receiver Module (UART0: GP0, GP1)**: Parses `$GPGGA` and `$GPRMC` strings to extract current latitude and longitude.
- **I2C Magnetometer Compass (GP4, GP5)**: Measures the vehicle's active heading (0 to 359°).
- **Steering Rudder Servo (GP10)**: Steers the vehicle (90° center, 45° left, 135° right).
- **Waypoint Cycle Button (GP13)**: Switches between three preloaded destination waypoints.
- **Active Buzzer (GP11)**: Beeps when a waypoint target is reached (< 5 meters).
- **16x2 I2C LCD (GP4, GP5 shared)**: Displays current heading, distance to target, and steering command.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| GPS Receiver Module | `gps` | Yes (represented by serial UART generator) | Yes (Neo-6M or similar) |
| HMC5883L I2C Compass | `compass` | Yes (represented by gyroscope/compass) | Yes |
| Servo Motor | `servo` | Yes | Yes (rudder steering) |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| GPS Module | TXD | GP1 (UART0 RX) | Yellow | GPS Tx to Pico Rx |
| GPS Module | RXD | GP0 (UART0 TX) | Orange | Pico Tx to GPS Rx |
| GPS Module | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| Rudder Servo | Control (PWM) | GP10 | Orange | Rudder control signal line |
| Rudder Servo | VCC / GND | 5V / GND | Red / Black | Servo power lines |
| Active Buzzer | VCC (+) | GP11 | Blue | Waypoint chime buzzer |
| Cycle Button | Terminal 1 | GP13 | White | Waypoint cycle button |
| HMC5883L Compass | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Ground reference |

> **Wiring tip:** Connect the GPS receiver TX line to GP1 (Pico RX). The I2C Compass and LCD display connect to GP4/GP5 in parallel. The servo is on GP10, and the buzzer is on GP11.

## Code
```python
from machine import Pin, PWM, UART, I2C
import utime, math
from machine_lcd import I2cLcd

# GPS UART0
gps_uart = UART(0, baudrate=9600, tx=Pin(0), rx=Pin(1))

# Steering Servo & Alarm Buzzer
steer_servo = PWM(Pin(10)); steer_servo.freq(50)
buzzer = Pin(11, Pin.OUT); buzzer.value(0)
btn_waypoint = Pin(13, Pin.IN, Pin.PULL_UP)

# I2C Bus 0: LCD & Magnetometer
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Waypoint Coordinates list: (Latitude, Longitude, Name)
WAYPOINTS = [
    (37.7749, -122.4194, "San Fran"),
    (34.0522, -118.2437, "Los Ang "),
    (40.7128, -74.0060,  "New York")
]
active_wp_idx = 0

# Telemetry state
curr_lat = 37.7740
curr_lon = -122.4190
current_heading = 0.0

def set_rudder(angle):
    """Aligns steering: 90 is straight, 45 is hard left, 135 is hard right."""
    angle = max(45, min(135, angle))
    duty = 1638 + int(angle * (8192 - 1638) / 180)
    steer_servo.duty_u16(duty)

def read_compass_heading():
    """Reads HMC5883L magnetometer registers (simplified simulator simulation)."""
    try:
        # HMC5883L address is 0x1E. Read 6 data bytes starting at register 0x03
        # In real hardware, configure configuration registers before reading.
        data = i2c.readfrom_mem(0x1E, 0x03, 6)
        x = (data[0] << 8) | data[1]
        z = (data[2] << 8) | data[3]
        y = (data[4] << 8) | data[5]
        if x > 32767: x -= 65536
        if y > 32767: y -= 65536
        
        # Calculate heading in radians, map to degrees
        heading = math.atan2(y, x) * 180 / math.pi
        if heading < 0:
            heading += 360
        return heading
    except OSError:
        # Fallback simulated drift
        global current_heading
        current_heading = (current_heading + 2) % 360
        return current_heading

def calculate_bearing_and_distance(lat1, lon1, lat2, lon2):
    """Calculates flat-earth distance (meters) and bearing (degrees) to target."""
    # Convert degrees to radians
    lat1, lon1, lat2, lon2 = map(math.radians, [lat1, lon1, lat2, lon2])
    
    # Distance approximation
    R = 6371000  # Earth radius in meters
    dlat = lat2 - lat1
    dlon = lon2 - lon1
    a = math.sin(dlat/2)**2 + math.cos(lat1)*math.cos(lat2)*math.sin(dlon/2)**2
    c = 2 * math.atan2(math.sqrt(a), math.sqrt(1-a))
    distance = R * c
    
    # Bearing calculation
    y = math.sin(dlon) * math.cos(lat2)
    x = math.cos(lat1)*math.sin(lat2) - math.sin(lat1)*math.cos(lat2)*math.cos(dlon)
    bearing = math.atan2(y, x) * 180 / math.pi
    bearing = (bearing + 360) % 360
    
    return distance, bearing

def parse_gps():
    """Reads UART buffer and searches for $GPRMC/$GPGGA sentence data."""
    global curr_lat, curr_lon
    if gps_uart.any():
        line = gps_uart.readline()
        try:
            # Decode to string and split CSV
            data = line.decode('utf-8').split(',')
            if data[0] == "$GPRMC" and data[2] == "A":  # Check status Active
                # Parse Latitude (DDMM.MMMM format)
                raw_lat = float(data[3])
                lat_deg = int(raw_lat / 100)
                lat_min = raw_lat - (lat_deg * 100)
                curr_lat = lat_deg + (lat_min / 60.0)
                if data[4] == "S": curr_lat = -curr_lat
                
                # Parse Longitude (DDDMM.MMMM format)
                raw_lon = float(data[5])
                lon_deg = int(raw_lon / 100)
                lon_min = raw_lon - (lon_deg * 100)
                curr_lon = lon_deg + (lon_min / 60.0)
                if data[6] == "W": curr_lon = -curr_lon
                
                print("GPS Fix: Lat:{:.4f} Lon:{:.4f}".format(curr_lat, curr_lon))
        except Exception:
            pass # Ignore malformed lines

set_rudder(90)  # Rudder straight
lcd.clear(); lcd.putstr("GPS Tracker\nNav System Ready")
utime.sleep(1.5)

print("GPS Waypoint Navigation Engine active.")
last_btn = 0

while True:
    now = utime.ticks_ms()
    
    # 1. Parse GPS and read compass
    parse_gps()
    heading = read_compass_heading()
    
    # 2. Get active target waypoint parameters
    target_lat, target_lon, target_name = WAYPOINTS[active_wp_idx]
    
    # 3. Calculate path metrics
    distance, target_bearing = calculate_bearing_and_distance(
        curr_lat, curr_lon, target_lat, target_lon
    )
    
    # 4. Check if waypoint reached (< 10 meters)
    if distance < 10.0:
        print(">> WAYPOINT REACHED: ", target_name)
        # Waypoint arrived chime
        for _ in range(3):
            buzzer.value(1); utime.sleep_ms(150); buzzer.value(0); utime.sleep_ms(50)
            
        # Cycle to next waypoint
        active_wp_idx = (active_wp_idx + 1) % len(WAYPOINTS)
        
    # Manual cycle button (GP13)
    if btn_waypoint.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        active_wp_idx = (active_wp_idx + 1) % len(WAYPOINTS)
        last_btn = now
        print("Cycled to waypoint:", WAYPOINTS[active_wp_idx][2])
        
    # 5. Closed-loop steering calculation
    # Calculate shortest angle deflection (-180 to 180)
    heading_error = target_bearing - heading
    if heading_error > 180: heading_error -= 360
    elif heading_error < -180: heading_error += 360
    
    # Proportional steering: rudder angle = 90 + error (with limits)
    # Scale error to rudder range (rudder angle offsets up to 45 degrees)
    rudder_cmd = 90 + int(heading_error * 0.5)
    set_rudder(rudder_cmd)
    
    # 6. Update display HUD
    lcd.clear()
    lcd.putstr("Tgt:{} H:{:.0f}\n".format(target_name[:8], heading))
    lcd.putstr("D:{:.0f}m Steer:{}".format(distance, rudder_cmd))
    
    print("Tgt:{} | Dist:{:.1f}m | Hdg:{:.0f} | Brg:{:.0f} | Steer:{}".format(
        target_name, distance, heading, target_bearing, rudder_cmd
    ))
    
    utime.sleep_ms(200)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **GPS Module**, **HMC5883L Compass** (gyroscope), **Rudder Servo**, **Buzzer**, **Button**, and **I2C LCD** onto the canvas.
2. Connect GPS TX to **GP1 (Rx)**, Servo to **GP10**, Buzzer to **GP11**, Button to **GP13**, and Compass + LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Modify the GPS simulator outputs to change coordinates. Observe the calculated distance decrease.
5. Move coordinates within 10 meters of San Francisco (37.7749, -122.4194). Verify that the buzzer beeps and the target cycles to Los Angeles.

## Expected Output
Terminal:
```
GPS Waypoint Navigation Engine active.
Tgt:San Fran | Dist:103.4m | Hdg:15 | Brg:20 | Steer:92
GPS Fix: Lat:37.7749 Lon:-122.4194
>> WAYPOINT REACHED:  San Fran
Tgt:Los Ang  | Dist:559030.1m | Hdg:17 | Brg:142 | Steer:152
```

## Expected Canvas Behavior
* Startup: OLED/LCD reads `GPS Tracker` / `Nav System Ready`. Rudder servo moves to 90°.
* Waypoint targets are displayed with active distance readouts.
* Adjusting the coordinates close to target: Rudder servo aligns back to 90°. When distance is reached, buzzer beeps and target shifts.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `math.atan2(y, x)` | Calculates the target vector bearing angle in radians relative to the current position. |
| `heading_error = target_bearing - heading` | Computes the deviation between the vehicle's current direction and target destination bearing. |

## Hardware & Safety Concept: Autonomous Vehicle Fail-safes
Autonomous vehicle computers rely on multiple subsystems (GPS, Compass, actuators) working in harmony. If the GPS lose signal locks (e.g. going through a tunnel), the latitude and longitude inputs stop updating, which would cause the navigation formulas to guide the vehicle in circles indefinitely. High-reliability navigation systems implement **GPS lock checks**: if no new valid NMEA frames are received within 5 seconds, the computer disables steering control, sounds the alarm, and enters a safe fail-stop state.

## Try This! (Challenges)
1. **Dynamic Navigation PID**: Replace the proportional rudder control logic with a complete PID feedback loop to prevent steering overcorrection.
2. **Reverse/Lost Alarm**: Sound the alarm yelp continuously if the distance to the target increases consecutively for more than 10 seconds (lost direction detection).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Rudder swings wildly back and forth | Heading error wrap error | Ensure the heading wrap logic is active: if `heading_error` is not bound within `[-180, 180]`, the rudder will steer the long way around, causing oscillation. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [127 - Pico Bluetooth Robot HC05 L298N LCD](../advanced/127-pico-bluetooth-robot-hc05-l298n-lcd.md)
- [197 - Pico Robotic Arm PID Position Controller](197-pico-robotic-arm-pid.md)
