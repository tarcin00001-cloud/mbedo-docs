# 75 - HC-SR04 Proximity Sensor Serial Logs

Measure physical distance to objects using an HC-SR04 ultrasonic sensor and print distance logs in centimeters to the Serial Monitor.

## Goal
Learn how to interface an ultrasonic sensor, trigger acoustic pulses, measure returned echo durations using `pulseIn()`, and convert time-of-flight measurements into distance.

## What You Will Build
An HC-SR04 sensor connects to the ARIES v3 board (Trig on GPIO 14, Echo on GPIO 15). The board triggers a measurement pulse every 500 ms, reads the echo response pulse width, and logs the calculated distance.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic_sensor` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 Sensor | VCC | 5V | Red | Power input |
| HC-SR04 Sensor | Trig | GPIO 14 | Orange | Trigger control line |
| HC-SR04 Sensor | Echo | GPIO 15 | Yellow | Echo signal line |
| HC-SR04 Sensor | GND | GND | Black | Ground reference |

> **Wiring tip:** The HC-SR04 is a 5V device. While the trigger pin works fine with 3.3V logic from the ARIES board, the Echo pin outputs 5V. On a physical board, use a resistor voltage divider (e.g. 1 kΩ and 2 kΩ) on the Echo pin to drop the voltage to 3.3V before connecting it to GPIO 15 to protect the SoC pin.

## Code
```cpp
// HC-SR04 Proximity Sensor Serial Logs
const int TRIG_PIN = 14;
const int ECHO_PIN = 15;

void setup() {
  Serial.begin(115200);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  // Set trigger line LOW to avoid floating pulses
  digitalWrite(TRIG_PIN, LOW);
  
  Serial.println("HC-SR04 Ultrasonic Sensor Initialized.");
}

void loop() {
  // Generate a clean 10-microsecond HIGH trigger pulse
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  // Measure the duration of the HIGH signal on the Echo pin (in microseconds)
  long duration = pulseIn(ECHO_PIN, HIGH);
  
  // Calculate distance: speed of sound = 343 m/s = 0.0343 cm/us
  // Distance = (Time * Speed) / 2 (accounting for round trip)
  float distance = (duration * 0.0343) / 2.0;
  
  if (duration == 0 || distance > 400.0) {
    Serial.println("Warning: Target Out of Range");
  } else {
    Serial.print("Distance: ");
    Serial.print(distance, 1);
    Serial.println(" cm");
  }
  
  delay(500); // 2Hz sampling frequency
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** and **HC-SR04 Sensor** components onto the canvas.
2. Wire the sensor: **Trig** to **GPIO 14**, **Echo** to **GPIO 15**, **VCC** to **5V**, and **GND** to **GND**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. Move the distance slider on the HC-SR04 widget and watch the output on the Serial Monitor.

## Expected Output
Serial Monitor:
```
HC-SR04 Ultrasonic Sensor Initialized.
Distance: 45.2 cm
Distance: 12.8 cm
Warning: Target Out of Range
```

## Expected Canvas Behavior
* The Serial Monitor logs updated distance measurements every 500 ms.
* Sliding the target slider on the ultrasonic widget immediately changes the distance value printed.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(TRIG_PIN, OUTPUT)` | Configures GPIO 14 as trigger output. |
| `digitalWrite(TRIG_PIN, HIGH)` | Drives trigger HIGH to start sending sonic pulses. |
| `pulseIn(ECHO_PIN, HIGH)` | Measures the time (in microseconds) until the Echo pin returns to LOW. |
| `(duration * 0.0343) / 2.0` | Converts time-of-flight to distance in centimeters. |

## Hardware & Safety Concept
* **Ultrasonic Time-of-Flight**: The HC-SR04 transmitter emits an 8-cycle sonic burst at 40 kHz. The receiver listens for the bounce. The Echo pin stays HIGH for the exact duration the sound wave travels to the target and back. By measuring this time and multiplying by the speed of sound, we find the round-trip distance. Dividing by 2 gives the single-trip distance to the obstacle.

## Try This! (Challenges)
1. **Inch Display**: Modify the code to calculate and print the distance in inches (1 inch = 2.54 cm).
2. **Moving Average Filter**: Save the last 3 measurements using individual state variables (no arrays) and print the averaged distance to filter out noise.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Readings are always "Out of Range" or 0 | Echo/Trig lines reversed | Swap the connections of GPIO 14 and GPIO 15. |
| Constant 0 distance on hardware | Missing pull-up/divider | Check voltage levels on Echo pin; ensure ground wire is shared. |
| Readings jump erratically | Sensor is measuring an angled surface | Ensure the sensor faces flat surfaces perpendicular to the sonic cone. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [76 - HC-SR04 Proximity Alarm](76-hc-sr04-proximity-alarm.md)
- [77 - HC-SR04 Distance Bar Graph LCD](77-hc-sr04-distance-bar-graph-lcd.md)
- [37 - IR Obstacle Sensor Digital Read](../beginner/37-ir-obstacle-sensor-digital-read.md)
