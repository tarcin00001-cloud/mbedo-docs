# 160 - ESP32 Ultrasonic Wind Anemometer

Build a solid-state ultrasonic wind anemometer that uses two opposing HC-SR04 ultrasonic sensors to calculate wind velocity based on the differences in sound wave flight times, displaying results on an LCD.

## Goal
Learn how to measure sub-millisecond time-of-flight differences, convert time parameters to velocities, and construct wind speed estimation algorithms.

## What You Will Build
Two HC-SR04 sensors are mounted facing each other at a fixed distance of 30 cm (0.3 meters). Sensor A is on GPIO 12/13, Sensor B on GPIO 14/15. A 16x2 LCD is on I2C. The ESP32 triggers both sensors, measures the flight times of sound pulses in both directions, calculates the speed difference caused by wind, and displays the wind speed in meters per second (m/s) on the LCD.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-SR04 Ultrasonic Sensors (2) | `ultrasonic_sensor` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Sensor A (Upwind) | Trig | GPIO12 | Orange | Upwind trigger |
| Sensor A (Upwind) | Echo | GPIO13 | Yellow | Upwind echo |
| Sensor B (Downwind)| Trig | GPIO14 | Green | Downwind trigger |
| Sensor B (Downwind)| Echo | GPIO15 | Blue | Downwind echo |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Opposing sensors | VCC / GND | 5V / GND | Red / Black | Shared power rails |

> **Wiring tip:** Opposing sensors are mounted facing each other on a rigid plastic pipe or frame exactly 30 cm apart. Wire the triggers to GPIO 12 and 14, and echo lines to GPIO 13 and 15.

## Code
```cpp
// Ultrasonic Wind Anemometer (ToF Difference calculation)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int TRIG_A = 12;
const int ECHO_A = 13;

const int TRIG_B = 14;
const int ECHO_B = 15;

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Physical dimensions
const float DISTANCE_M = 0.30;       // Distance between sensors in meters (30 cm)
const float SPEED_OF_SOUND = 343.0; // Speed of sound at 20 °C in m/s

unsigned long lastSampleTime = 0;
const unsigned long SAMPLE_INTERVAL_MS = 1000; // Sample once per second

// Read raw pulse time in microseconds
long getPulseTime(int trigPin, int echoPin) {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  
  // 10ms timeout (sound travels ~3.4m, path is 0.3m)
  long duration = pulseIn(echoPin, HIGH, 10000); 
  return duration;
}

void setup() {
  Serial.begin(115200);
  
  pinMode(TRIG_A, OUTPUT);
  pinMode(ECHO_A, INPUT);
  pinMode(TRIG_B, OUTPUT);
  pinMode(ECHO_B, INPUT);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Anemometer ready");
  lcd.setCursor(0, 1);
  lcd.print("Calibrating...");
  
  delay(1500);
  lcd.clear();
}

void loop() {
  unsigned long now = millis();
  
  if (now - lastSampleTime >= SAMPLE_INTERVAL_MS) {
    // 1. Measure flight time from A to B (Upwind path)
    long tofA_us = getPulseTime(TRIG_A, ECHO_A);
    delay(20); // Let echo reflections decay
    
    // 2. Measure flight time from B to A (Downwind path)
    long tofB_us = getPulseTime(TRIG_B, ECHO_B);
    
    if (tofA_us > 0 && tofB_us > 0) {
      // Convert microseconds to seconds
      float tA = (float)tofA_us / 1000000.0;
      float tB = (float)tofB_us / 1000000.0;
      
      // Ultrasonic Anemometer Formula:
      // Wind speed Vw = (d / 2) * (1/tA - 1/tB)
      // Positive result indicates wind blowing from A to B
      float windSpeed = (DISTANCE_M / 2.0) * ((1.0 / tA) - (1.0 / tB));
      
      // Limit speed output for minor noise deviations
      if (abs(windSpeed) < 0.1) windSpeed = 0.0;
      
      // Update LCD
      lcd.setCursor(0, 0);
      lcd.print("Wind Spd: ");
      lcd.print(abs(windSpeed), 2);
      lcd.print(" m/s ");
      
      lcd.setCursor(0, 1);
      lcd.print("Dir: ");
      if (windSpeed > 0.1) {
        lcd.print("A -> B (North)");
      } else if (windSpeed < -0.1) {
        lcd.print("B -> A (South)");
      } else {
        lcd.print("CALM          ");
      }
      
      Serial.print("ToF A: "); Serial.print(tofA_us);
      Serial.print(" us | ToF B: "); Serial.print(tofB_us);
      Serial.print(" us | Wind Speed: "); Serial.print(windSpeed, 2); Serial.println(" m/s");
    } else {
      lcd.setCursor(0, 0);
      lcd.print("Sensor Offline  ");
      Serial.println("Error reading flight times.");
    }
    
    lastSampleTime = now;
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, two **HC-SR04 Sensors**, and **16x2 I2C LCD** onto the canvas.
2. Wire Sensor A to **GPIO12/GPIO13**, Sensor B to **GPIO14/GPIO15**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the distance widget of Sensor A lower than Sensor B. Watch the wind speed update and display direction on the LCD.

## Expected Output
Serial Monitor:
```
ToF A: 874 us | ToF B: 874 us | Wind Speed: 0.00 m/s
ToF A: 850 us | ToF B: 900 us | Wind Speed: 9.80 m/s
```

LCD Display (wind blowing):
```
Wind Spd: 9.80 m/s
Dir: A -> B (North)
```

## Expected Canvas Behavior
* If both simulated sensors are set to the same distance (e.g. 30 cm, representing ~874 microseconds), the LCD displays "Wind Spd: 0.00 m/s / Dir: CALM".
* Offsetting the sliders (simulating sound acceleration or deceleration) calculates a corresponding wind velocity and shows the direction of the wind (A to B or B to A) on the screen.

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `tofA_us / 1000000.0` | Converts microsecond time-of-flight integers to decimal seconds. |
| `((1.0 / tA) - (1.0 / tB))` | Computes the differences between reciprocal velocities. |
| `DISTANCE_M / 2.0` | Scales the result based on the physical distance between the transducers. |

## Hardware & Safety Concept: Solid-state Anemometers and Time-of-Flight
Traditional cup anemometers use moving parts that wear out or freeze in cold weather. Ultrasonic anemometers have no moving parts. They calculate wind speed using the speed of sound: sound waves traveling with the wind are accelerated, while waves traveling against the wind are slowed down. Measuring this time difference allows calculating wind speed and direction.

## Try This! (Challenges)
1. **Temperature compensation**: The speed of sound changes with temperature (\(c = 331.3 + 0.6 \cdot T\)). Add a DHT22 sensor (Project 71) to dynamically correct the speed of sound in the calculations.
2. **Interactive speed unit toggle**: Add a button on GPIO 25 that toggles the displayed wind speed units between meters per second (m/s) and knots (kt) (1 m/s = 1.943 kt).
3. **Gust logger to SD card**: Add an SD card module (Project 136) to log peak wind gusts to a CSV file.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Wind speed fluctuates wildly when still | Air currents or sensor noise | Average 10 readings to smooth out turbulence |
| Reading is always "Sensor Offline" | Echo signals not received | Verify that both sensors are pointing directly at each other and wired correctly |
| Calculated speed is incorrect | Incorrect distance setting | Update the `DISTANCE_M` constant in the code to match the physical distance between the sensors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [75 - ESP32 Proximity Sensor Serial Logs](../intermediate/75-esp32-hcsr04-proximity-sensor-serial-logs.md)
- [139 - ESP32 Multi-Sensor Environmental HUD](139-esp32-multi-sensor-environmental-hud.md)
- [150 - ESP32 Dual-protocol Telemetry Logger](150-esp32-dual-protocol-telemetry-logger.md)
