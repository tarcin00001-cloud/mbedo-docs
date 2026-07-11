# 200 - ESP32 Smart GPS Autonomous Waypoint Tracker

Build an autonomous navigation vehicle controller on the ESP32 that parses current location coordinates from a NEO-6M GPS receiver, measures direction headings using an HMC5883L magnetometer compass, calculates target bearings to a destination waypoint, and drives a steering rudder servo.

## Goal
Learn how to compute distances and bearings between coordinates (Haversine navigation math), calculate steering errors, and control steering servos.

## What You Will Build
A NEO-6M GPS is connected to UART2 (RX2: 16, TX2: 17). An HMC5883L magnetometer is on I2C (GPIO 21/22). A steering rudder servo is on GPIO 27, and a 16x2 LCD on I2C. The ESP32 calculates the distance and target bearing to a preset waypoint. It compares the target bearing to the current compass heading, calculating the steering error to steer the servo to guide the vehicle to the destination.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| NEO-6M GPS Module | `gps_neo6m` | Yes | Yes |
| HMC5883L Magnetometer | `magnetometer` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| External Power Supply (5V, 2A) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NEO-6M Module | TXD / RXD | GPIO16 / GPIO17 | Green / Yellow | GPS Serial lines |
| HMC5883L | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (Shared) |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (Shared) |
| Servo Motor | Signal | GPIO27 | Brown | Steering rudder servo |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Power the steering servo from the 5V Vin rail.

## Code
```cpp
// Smart GPS Autonomous Waypoint Tracker (Compass + GPS + Servo steering)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <TinyGPS++.h>
#include <ESP32Servo.h>
#include <math.h>

#define RX2_PIN 16
#define TX2_PIN 17
#define HMC5883L_ADDR 0x1E
#define SERVO_PIN 27

TinyGPSPlus gps;
LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo steerServo;

// Target Waypoint Coordinates
// e.g. Central Park, NY (replace with your coordinates)
const double TARGET_LAT = 40.785091; 
const double TARGET_LNG = -73.968285;

// Steering configuration
const int SERVO_CENTER = 90; // Rudder center (straight)
const float STEER_GAIN = 1.2; // Gain factor for steering response

// Declination angle at your location (in radians)
const float DECLINATION_ANGLE = -0.22; // e.g. New York declination

void initHMC5883L() {
  Wire.beginTransmission(HMC5883L_ADDR);
  Wire.write(0x02); // Mode register
  Wire.write(0x00); // Continuous measurement
  Wire.endTransmission();
}

void readMagnetometer(int &x, int &y, int &z) {
  Wire.beginTransmission(HMC5883L_ADDR);
  Wire.write(0x03); // Data registers MSB
  Wire.endTransmission();
  
  Wire.requestFrom(HMC5883L_ADDR, 6);
  if (Wire.available() >= 6) {
    x = (Wire.read() << 8) | Wire.read();
    y = (Wire.read() << 8) | Wire.read();
    z = (Wire.read() << 8) | Wire.read();
  }
}

float getCompassHeading() {
  int mx, my, mz;
  readMagnetometer(mx, my, mz);
  
  float heading = atan2((float)my, (float)mx);
  heading += DECLINATION_ANGLE;
  
  if (heading < 0) heading += 2 * M_PI;
  if (heading > 2 * M_PI) heading -= 2 * M_PI;
  
  return heading * 180.0 / M_PI; // Return heading in degrees
}

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, RX2_PIN, TX2_PIN); // GPS Serial
  
  steerServo.attach(SERVO_PIN);
  steerServo.write(SERVO_CENTER);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("GPS Waypoint Nav");
  lcd.setCursor(0, 1);
  lcd.print("Initializing...");
  
  initHMC5883L();
  delay(1500);
  lcd.clear();
}

void loop() {
  // 1. Feed GPS parser
  while (Serial2.available() > 0) {
    gps.encode(Serial2.read());
  }
  
  // Get current heading from compass
  float currentHeading = getCompassHeading();
  
  if (gps.location.isValid()) {
    double currentLat = gps.location.lat();
    double currentLng = gps.location.lng();
    
    // 2. Calculate Distance and Target Bearing
    // Distance in meters
    double distanceToTarget = TinyGPSPlus::distanceBetween(currentLat, currentLng, TARGET_LAT, TARGET_LNG);
    // Bearing in degrees (0 to 360)
    double targetBearing = TinyGPSPlus::courseTo(currentLat, currentLng, TARGET_LAT, TARGET_LNG);
    
    // 3. Calculate Steering Error (Target Bearing - Current Heading)
    float headingError = targetBearing - currentHeading;
    
    // Adjust error to range -180 to 180 degrees
    if (headingError < -180.0) headingError += 360.0;
    if (headingError > 180.0)  headingError -= 360.0;
    
    // 4. Calculate Rudder Servo Angle
    // If headingError > 0 -> steer right; if < 0 -> steer left
    int servoAngle = SERVO_CENTER + (int)(headingError * STEER_GAIN);
    servoAngle = constrain(servoAngle, 30, 150); // Constrain steering limits
    
    // Command Rudder
    // If close to target (within 3 meters), stop steering
    if (distanceToTarget < 3.0) {
      steerServo.write(SERVO_CENTER);
      Serial.println("!!! WAYPOINT DESTINATION ARRIVED !!!");
    } else {
      steerServo.write(servoAngle);
    }
    
    // Update LCD Screen
    lcd.setCursor(0, 0);
    lcd.print("Dist: "); lcd.print(distanceToTarget, 1); lcd.print("m   ");
    lcd.setCursor(0, 1);
    lcd.print("H:"); lcd.print((int)currentHeading);
    lcd.print(" Brg:"); lcd.print((int)targetBearing);
    lcd.print("   ");
    
    Serial.print("Dist: "); Serial.print(distanceToTarget, 1);
    Serial.print(" m | Bear: "); Serial.print(targetBearing, 0);
    Serial.print(" | Head: "); Serial.print(currentHeading, 0);
    Serial.print(" | Error: "); Serial.print(headingError, 1);
    Serial.print(" | Servo: "); Serial.println(servoAngle);
  } else {
    // No GPS fix
    lcd.setCursor(0, 0);
    lcd.print("GPS: SEARCHING  ");
    lcd.setCursor(0, 1);
    lcd.print("Heading: "); lcd.print((int)currentHeading); lcd.print("    ");
    
    steerServo.write(SERVO_CENTER); // Hold rudder straight
    
    delay(200);
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **NEO-6M GPS**, **HMC5883L**, **Servo**, and **16x2 I2C LCD** onto the canvas.
2. Wire components: GPS to UART2, I2C to **GPIO21/GPIO22**, and Servo to **GPIO27**.
3. Paste the code and click **Run**.
4. Slide the GPS coordinates to a location near the preset target waypoint. Set satellites > 4.
5. Slide the HMC5883L compass angle slider. Watch the steering servo rotate to correct heading errors and guide the vehicle.

## Expected Output
Serial Monitor:
```
Dist: 45.2 m | Bear: 180 | Head: 160 | Error: 20.0 | Servo: 114
Dist: 12.4 m | Bear: 180 | Head: 195 | Error: -15.0 | Servo: 72
!!! WAYPOINT DESTINATION ARRIVED !!!
```

LCD Display:
```
Dist: 12.4m
H:195 Brg:180
```

## Expected Canvas Behavior
* At boot, if the GPS has no fix, the LCD displays `GPS: SEARCHING`. The servo widget is at 90°.
* Once the GPS coordinates are valid, the LCD displays the distance and bearing.
* Adjusting the simulated compass slider rotates the servo widget to correct the heading error.
* If the coordinates slider is moved within 3 meters of the target, the servo returns to 90° (rudder straight) and the Serial Monitor prints `WAYPOINT DESTINATION ARRIVED`.

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `distanceBetween(...)` | Calculates the distance in meters between coordinates using the Haversine formula. |
| `courseTo(...)` | Calculates the target bearing angle relative to North (0°–360°). |
| `headingError += 360` | Normalizes heading error to the shortest rotation path (-180° to 180°). |
| `steerServo.write(...)` | Commands the steering servo to correct the heading error. |

## Hardware & Safety Concept: Autonomous Waypoint Steering and Haversine Navigation
Autonomous vehicle navigation systems (like rovers or boats) combine GPS coordinates and compass headings. The GPS calculates the current position and target bearing to the destination waypoint. The compass measures the current heading. Comparing the bearing and heading gives the **heading error**. The controller uses this error to steer the rudder or wheels, aligning the vehicle's heading with the target bearing until it arrives.

## Try This! (Challenges)
1. **Multi-Waypoint Path routing**: Program a list of multiple waypoints, automatically switching to the next waypoint once the vehicle arrives.
2. **SD Card Voyage Logger**: Log coordinates, headings, and steering commands to an SD card (Project 147) throughout the voyage.
3. **Emergency Sonar brake**: Integrate an HC-SR04 sonar sensor (Project 198) to stop the vehicle if an obstacle is detected in its path.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Steering servo rotates to limits and stays stuck | Declination angle or axis sign error | Verify that the magnetometer declination angle is correct for your location, and check the `atan2` axis assignments |
| Rudder steers in the opposite direction of error | Steering direction inverted | Change the sign of the gain factor in the servo calculation (e.g. change `+ error` to `- error`) |
| No GPS lock | GPS antenna shielded | Check coordinates settings and ensure the satellite count slider is turned up |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [145 - ESP32 GPS Logger](145-esp32-gps-logger.md)
- [146 - ESP32 GPS Location Display](146-esp32-gps-location-display.md)
- [149 - ESP32 I2C OLED Tilt-compensated Compass](149-esp32-i2c-oled-tilt-compensated-compass.md)
