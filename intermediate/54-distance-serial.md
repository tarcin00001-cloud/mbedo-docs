# 54 - Distance Serial

Measure and print real-time distance readouts in centimeters using an HC-SR04 Ultrasonic Sensor.

## Goal
Learn how to generate high-frequency trigger pulses, measure echo duration using the built-in `pulseIn()` function, and calculate physical distance based on the speed of sound.

## What You Will Build
The Arduino triggers the ultrasonic sensor, measures the time it takes for the echo to return, calculates the distance in centimeters, and prints it to the Terminal every 200 ms.

**Why D2 and D3?** Pin D2 is configured as a digital output to send the trigger pulse. Pin D3 is configured as a digital input to measure the echo pulse duration.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-SR04 Ultrasonic | `hc_sr04` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 Sensor | VCC | 5V | Power supply (5V) |
| HC-SR04 Sensor | TRIG | D2 | Trigger signal output from Arduino |
| HC-SR04 Sensor | ECHO | D3 | Echo pulse input to Arduino |
| HC-SR04 Sensor | GND | GND | Ground reference |

## Code
```cpp
const int TRIG_PIN = 2;
const int ECHO_PIN = 3;

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  Serial.begin(9600);
  Serial.println("HC-SR04 Distance Sensor Ready");
}

void loop() {
  // Clear the trigger pin
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  
  // Send a 10 microsecond HIGH pulse to trigger measurement
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  // Read the echo pin pulse duration (in microseconds)
  long duration = pulseIn(ECHO_PIN, HIGH);
  
  // Calculate distance in centimeters:
  // Speed of sound is 340 m/s or 0.034 cm/microsecond.
  // We divide by 2 because the wave travels to the object and back.
  long distance = duration * 0.034 / 2;
  
  Serial.print("Distance: ");
  Serial.print(distance);
  Serial.println(" cm");
  
  delay(200); // Wait 200 ms between measurements
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **HC-SR04 Ultrasonic Sensor** onto the canvas.
2. Connect HC-SR04 **VCC** to Arduino **5V**, **TRIG** to Arduino **D2**, **ECHO** to Arduino **D3**, and **GND** to Arduino **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Double-click the HC-SR04 sensor on the canvas, adjust the distance slider, and watch the values print in the Terminal.

## Expected Output

Terminal:
```
HC-SR04 Distance Sensor Ready
Distance: 120 cm
Distance: 45 cm
Distance: 250 cm
...
```

### Expected Canvas Behavior

| Distance Slider Input | raw Echo duration | Calculated Distance |
| --- | --- | --- |
| 100 cm | 5882 microseconds | "Distance: 100 cm" |
| 20 cm | 1176 microseconds | "Distance: 20 cm" |

The printed output tracks the slider position in real-time.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `digitalWrite(TRIG_PIN, HIGH); delayMicroseconds(10);` | Generates the exact 10-microsecond trigger pulse required to wake the sensor and launch its ultrasonic burst. |
| `pulseIn(ECHO_PIN, HIGH)` | Measures how long pin D3 remains HIGH, which represents the flight time of the sound waves in microseconds. |
| `duration * 0.034 / 2` | Converts flight time to distance in centimeters based on the speed of sound. |

## Hardware & Safety Concept: How Ultrasonic Ranging Works
The **HC-SR04** sensor works on the principle of echolocation (similar to bats or sonar systems).
1. When triggered, the transmitter transducer sends out an 8-cycle sonic burst at **40 kHz** (well above human hearing).
2. The sound waves travel through the air, bounce off a nearby surface, and return.
3. The receiver transducer detects the echo and drives the `ECHO` pin HIGH.
4. The duration that the `ECHO` pin remains HIGH corresponds directly to the total travel time of the sound wave.

## Try This! (Challenges)
1. **Inch Conversion**: Modify the code to calculate and display the distance in inches instead of centimeters. (Hint: divide the centimeter value by `2.54` or multiply duration by `0.0133 / 2`).
2. **Out of Range Filter**: If the calculated distance exceeds `400` cm (the physical limit of the sensor) or is `0`, print "Out of range!" instead of the distance number.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Distance reading is constantly 0 | Trigger or Echo pin swapped | Verify that TRIG is connected to D2 and ECHO is connected to D3. Check that the code matches. |
| Reading bounces wildly | Sensor target is too small or acoustic reflection | Ensure the sensor is facing a flat, solid surface. Soft fabrics absorb ultrasonic waves and produce false readings. |

## Mode Notes
The built-in `pulseIn()` timing alias is parsed and supported by the MbedO interpreted mode runtime.

## Related Projects
- [15 - Analog Meter Serial](../beginner/15-analog-meter-serial.md)
- [55 - Distance Alert LED](55-distance-alert-led.md)
- [56 - Parking Beeper](56-parking-beeper.md)
