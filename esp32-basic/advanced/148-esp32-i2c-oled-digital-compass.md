# 148 - ESP32 I2C OLED Digital Compass

Build an electronic digital compass that reads magnetic field data from an HMC5883L 3-axis magnetometer, calculates cardinal headings (N, S, E, W), and renders a dynamic graphical pointer needle on an SSD1306 OLED screen.

## Goal
Learn how to compute heading angles using arctangent trigonometric functions (`atan2`), calibrate magnetic offsets, and draw rotatable needle lines on graphics screens.

## What You Will Build
An HMC5883L magnetometer and an SSD1306 OLED screen are connected to the I2C bus (GPIO 21/22). The ESP32 reads raw magnetic values on the X and Y axes, calculates the heading angle relative to magnetic North, and displays the angle, cardinal direction, and a line vector pointing North.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HMC5883L 3-Axis Magnetometer | `magnetometer` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HMC5883L | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| HMC5883L | VCC / GND | 3V3 / GND | Red / Black | Sensor power |
| SSD1306 OLED | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| SSD1306 OLED | VCC / GND | 3V3 / GND | Red / Black | OLED power |

> **Wiring tip:** Share the I2C SDA and SCL lines. Mount the HMC5883L flat on a non-magnetic breadboard (avoiding steel pins or magnets) to prevent magnetic interference.

## Code
```cpp
// I2C OLED Digital Compass (HMC5883L + OLED)
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <math.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// Simulated HMC5883L Register Addresses
#define HMC5883L_ADDR 0x1E
#define REG_MODE 0x02
#define REG_DATA_X_MSB 0x03

// Declination angle at your location (in radians)
// Look up your declination at http://www.magnetic-declination.com/
// e.g. Bangalore, India is +1 deg 20 min East -> 1.33 degrees -> 0.023 radians
const float DECLINATION_ANGLE = 0.023; 

void initHMC5883L() {
  Wire.beginTransmission(HMC5883L_ADDR);
  Wire.write(REG_MODE);
  Wire.write(0x00); // Continuous measurement mode
  Wire.endTransmission();
}

void readMagnetometer(int &x, int &y, int &z) {
  Wire.beginTransmission(HMC5883L_ADDR);
  Wire.write(REG_DATA_X_MSB);
  Wire.endTransmission();
  
  Wire.requestFrom(HMC5883L_ADDR, 6);
  if (Wire.available() >= 6) {
    x = (Wire.read() << 8) | Wire.read();
    y = (Wire.read() << 8) | Wire.read();
    z = (Wire.read() << 8) | Wire.read();
  }
}

String getCardinal(float degrees) {
  if (degrees >= 337.5 || degrees < 22.5)   return "N";
  if (degrees >= 22.5  && degrees < 67.5)   return "NE";
  if (degrees >= 67.5  && degrees < 112.5)  return "E";
  if (degrees >= 112.5 && degrees < 157.5)  return "SE";
  if (degrees >= 157.5 && degrees < 202.5)  return "S";
  if (degrees >= 202.5 && degrees < 247.5)  return "SW";
  if (degrees >= 247.5 && degrees < 292.5)  return "W";
  if (degrees >= 292.5 && degrees < 337.5)  return "NW";
  return "N";
}

void setup() {
  Serial.begin(115200);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED init failed!");
    while(1) {}
  }
  
  initHMC5883L();
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(10, 20);
  display.print("Compass Calibration");
  display.display();
  delay(1000);
}

void loop() {
  int rawX, rawY, rawZ;
  readMagnetometer(rawX, rawY, rawZ);
  
  // Calculate heading in radians using Y and X axis values
  float heading = atan2((float)rawY, (float)rawX);
  
  // Add declination angle correction
  heading += DECLINATION_ANGLE;
  
  // Correct for sign rollovers
  if (heading < 0) heading += 2 * M_PI;
  if (heading > 2 * M_PI) heading -= 2 * M_PI;
  
  // Convert to degrees
  float headingDegrees = heading * 180.0 / M_PI;
  
  String cardinal = getCardinal(headingDegrees);
  
  // Draw Compass Graphical Needle
  display.clearDisplay();
  
  // Draw outer dial ring (circle centered at 96, 32 with radius 25)
  int dialX = 96;
  int dialY = 32;
  int radius = 24;
  display.drawCircle(dialX, dialY, radius, SSD1306_WHITE);
  
  // Calculate needle pointer coordinates
  // Negate angle to rotate needle in opposite direction of heading turns
  float needleAngle = -heading; 
  int needleX = dialX + (int)(radius * sin(needleAngle));
  int needleY = dialY - (int)(radius * cos(needleAngle));
  
  // Draw needle line (from center to outside point)
  display.drawLine(dialX, dialY, needleX, needleY, SSD1306_WHITE);
  display.fillCircle(dialX, dialY, 2, SSD1306_WHITE); // Center cap
  
  // Draw Text HUD
  display.setCursor(0, 5);
  display.setTextSize(1);
  display.print("COMPASS");
  
  display.setTextSize(2);
  display.setCursor(0, 20);
  display.print(cardinal);
  
  display.setTextSize(1);
  display.setCursor(0, 45);
  display.print("Head: ");
  display.print((int)headingDegrees);
  display.print((char)247); // Degree symbol
  
  display.display();
  
  Serial.print("Heading: "); Serial.print(headingDegrees);
  Serial.print(" | Cardinal: "); Serial.println(cardinal);
  
  delay(150); // 7Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **HMC5883L**, and **SSD1306 OLED** onto the canvas.
2. Wire both I2C lines to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Adjust the magnetic angle slider on the HMC5883L widget. Watch the compass needle line rotate on the OLED screen.

## Expected Output
Serial Monitor:
```
Heading: 180.2 | Cardinal: S
Heading: 90.5 | Cardinal: E
Heading: 359.1 | Cardinal: N
```

OLED Screen Layout:
```
COMPASS    ┌───┐
           │ \ │ (Needle pointer rotating inside dial ring)
N          └───┘
Head: 359°
```

## Expected Canvas Behavior
* The OLED widget draws a circle representing the compass housing dial.
* Adjusting the magnetic heading slider on the simulated magnetometer rotates the needle line inside the circle widget dynamically.
* The cardinal direction changes as the slider moves.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `atan2(rawY, rawX)` | Computes the principal angle of the vector (X, Y) in radians between \(-\pi\) and \(\pi\). |
| `headingDegrees = heading * 180.0 / M_PI` | Converts the radian angle to a standard 0°–360° degree scale. |
| `needleX = dialX + radius * sin(needleAngle)` | Applies polar coordinate conversion (\(x = r \cdot \sin(\theta)\)) to plot the needle line tip. |

## Hardware & Safety Concept: Magnetometer Calibration and declination
Magnetometers measure the Earth's weak magnetic fields (approx 0.5 Gauss). Any nearby metal objects (steel screws, batteries) or electromagnetic fields (motors, power wires) distort these readings, creating **hard-iron** and **soft-iron** offsets. Compasses require manual calibration routines (rotating the sensor in circles to identify offsets) and must factor in **magnetic declination** (the angle offset between True Geographic North and Magnetic North at your specific location).

## Try This! (Challenges)
1. **Calibration Mode offset log**: Implement a routine that scans min/max readings on startup to calculate X/Y offsets automatically.
2. **Cardinal ticks indicator**: Draw small ticks representing North (N), South (S), East (E), and West (W) around the outer ring dial.
3. **Off-Course Alarm**: Sound a buzzer (GPIO 15) if the compass heading drifts more than 20 degrees away from a target travel path.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Compass needle is static or points in only one quadrant | Sensor registers locked | Verify HMC5883L continuous mode register setup (REG_MODE write 0x00) |
| Heading is inverted | Axis orientation mismatched | Invert the sign of the Y reading or swap axes inside the `atan2` function |
| Needles jump wildly | Interference from motors or laptops | Keep the sensor away from magnets and laptops; apply a running average filter to inputs |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [94 - ESP32 MPU-6050 Gyroscope Position HUD](../intermediate/94-esp32-mpu6050-gyroscope-position-hud.md)
- [60 - ESP32 OLED SSD1306 Display Setup](../intermediate/60-esp32-oled-ssd1306-display-setup.md)
- [149 - ESP32 I2C OLED Tilt-compensated Compass](149-esp32-oled-tilt-compensated-compass.md) (Next project)
