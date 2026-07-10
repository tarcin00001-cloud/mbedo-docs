# 75 - ESP32 HC-SR04 Proximity Sensor Serial Logs

Measure distance to objects using an HC-SR04 ultrasonic sensor and print structured distance logs to the Serial Monitor.

## Goal
Learn how to control trigger pulses, measure return echo timing using `pulseIn()`, and convert sound travel time into physical distances in centimeters.

## What You Will Build
An HC-SR04 ultrasonic sensor connected to GPIO 5 (Trig) and GPIO 18 (Echo). The ESP32 triggers ultrasonic bursts, measures the pulse response time, calculates the distance, and logs the value to the Serial Monitor every 500 ms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic_sensor` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 Sensor | VCC | 5V (Vin) | Red | Sensor requires 5V supply |
| HC-SR04 Sensor | Trig | GPIO5 | Orange | Trigger output pin |
| HC-SR04 Sensor | Echo | GPIO18 | Yellow | Echo input pin (3.3V voltage divider recommended) |
| HC-SR04 Sensor | GND | GND | Black | Ground reference |

> **Wiring tip:** The HC-SR04 runs on **5V** and outputs a 5V Echo pulse. The ESP32 pins are not 5V tolerant. In physical hardware builds, place a resistor voltage divider (e.g. 1 kΩ in series with Echo, and 2 kΩ to GND) to drop the 5V Echo output to a safe 3.3V logic level before connecting to GPIO 18.

## Code
```cpp
// HC-SR04 Proximity Sensor Serial Logs
const int TRIG_PIN = 5;
const int ECHO_PIN = 18;

void setup() {
  Serial.begin(115200);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  digitalWrite(TRIG_PIN, LOW);
  
  Serial.println("HC-SR04 Proximity Sensor ready.");
}

void loop() {
  // Clear the Trig pin
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  
  // Trigger the sensor by sending a 10 microsecond pulse
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  // Read Echo pulse travel duration in microseconds
  long duration = pulseIn(ECHO_PIN, HIGH);
  
  // Calculate distance in cm:
  // Speed of sound = 340 m/s = 0.034 cm/us.
  // Distance = (Time * Speed) / 2 (accounting for round trip)
  float distance = (duration * 0.0343) / 2.0;
  
  Serial.print("Echo Duration: ");
  Serial.print(duration);
  Serial.print(" us  |  Distance: ");
  
  // Filter out invalid/out-of-range sensor readings
  if (distance >= 400.0 || distance <= 2.0) {
    Serial.println("Out of range / No object detected");
  } else {
    Serial.print(distance, 1);
    Serial.println(" cm");
  }
  
  delay(500);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **HC-SR04 Sensor** onto the canvas.
2. Wire Trig to **GPIO5** and Echo to **GPIO18**.
3. Paste the code and click **Run**.
4. Adjust the distance slider on the HC-SR04 widget and watch the output print to the Serial Monitor.

## Expected Output
Serial Monitor:
```
HC-SR04 Proximity Sensor ready.
Echo Duration: 583 us  |  Distance: 10.0 cm
Echo Duration: 2915 us  |  Distance: 50.0 cm
Echo Duration: 0 us  |  Distance: Out of range / No object detected
```

## Expected Canvas Behavior
* The Serial Monitor prints updated measurements at a 2 Hz frequency.
* Adjusting the distance slider on the virtual sensor widget immediately changes the distance and duration printed.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `delayMicroseconds(10)` | Sends the required 10 µs trigger pulse to start the measurement cycle. |
| `pulseIn(ECHO_PIN, HIGH)` | Measures how long (in microseconds) the Echo pin stays HIGH. |
| `(duration * 0.0343) / 2.0` | Converts time duration to distance in centimeters. |
| `distance >= 400.0` | Filters out values above the sensor's maximum rating of ~4 meters. |

## Hardware & Safety Concept: Time-of-Flight Ranging
Ultrasonic distance sensors work on the **time-of-flight (ToF)** principle. The transmitter emits a high-frequency (40 kHz) sound burst. The sound wave travels through air, hits an object, and bounces back to the receiver. The sensor drives the Echo pin HIGH from the moment it transmits until it receives the reflection. Because the speed of sound is constant in air at a given temperature (~343 m/s at 20 °C), distance is proportional to time.

## Try This! (Challenges)
1. **Inch Converter**: Add code to calculate and display the distance in inches (1 inch = 2.54 cm).
2. **Speed of Sound Compensation**: Integrate a DHT22 sensor (Project 71) to calculate the speed of sound dynamically based on air temperature: \(v = 331.3 + 0.606 \times T\) m/s.
3. **Safety Brake Trigger**: Print a warning message on the Serial Monitor if the object gets closer than 15 cm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Distance always reads 0 | Echo pin not triggering | Verify Echo is wired to GPIO 18, check voltage divider resistors |
| Constant "Out of range" message | Sensor pointed at sound-absorbing soft objects (e.g. carpet) | Use a solid, flat surface to test reflections |
| Inaccurate measurements | Unstable power supply | Ensure VCC connects to the 5V/Vin rail, not 3V3 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [76 - ESP32 HC-SR04 Proximity Alarm](76-esp32-hcsr04-proximity-alarm.md)
- [77 - ESP32 HC-SR04 Distance Bar Graph LCD](77-esp32-hcsr04-distance-bar-graph-lcd.md)
- [101 - ESP32 Automatic Gate](../advanced/101-automatic-gate.md)
