# 75 - Pico Ultrasonic Serial

Measure and log target distances using an HC-SR04 Ultrasonic Proximity Sensor.

## Goal
Learn how to generate trigger pulses, measure echo return times, and calculate target distances using digital pins on the Raspberry Pi Pico.

## What You Will Build
A digital tape measure:
- **HC-SR04 Sensor (Trigger GP14, Echo GP15)**: Emits high-frequency ultrasonic bursts.
- **Serial Monitor**: Prints the calculated distance to the target object in centimeters (e.g. "Distance: 45 cm") every 500 ms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (voltage divider) |
| 4.7k-ohm Resistor | `resistor` | Optional | Yes (voltage divider) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 | VCC | 5V (or VBUS) | 5V required for sensor logic |
| HC-SR04 | Trig (Trigger) | GP14 | Trigger output |
| HC-SR04 | Echo | GP15 | Echo input (requires voltage divider) |
| HC-SR04 | GND | GND | Ground reference |

## Code
```cpp
const int TRIG_PIN = 14;
const int ECHO_PIN = 15;

void setup() {
  Serial.begin(9600);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  // Set trigger LOW to start
  digitalWrite(TRIG_PIN, LOW);
  Serial.println("Ultrasonic Distance Monitor Online");
}

void loop() {
  // 1. Generate 10 microsecond trigger pulse
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  // 2. Measure high pulse duration on Echo pin (microseconds)
  long duration = pulseIn(ECHO_PIN, HIGH);

  // 3. Calculate distance: duration * speed of sound (0.0343 cm/us) / 2
  int distance = duration * 0.0343 / 2;

  Serial.print("Distance: ");
  Serial.print(distance);
  Serial.println(" cm");

  delay(500); // Sample twice per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **HC-SR04 Ultrasonic Sensor** onto the canvas.
2. Connect HC-SR04: **VCC** to **5V**, **Trig** to **GP14**, **Echo** to **GP15**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the target obstacle distance slider on the HC-SR04 model and check the Serial logs.

## Expected Output

Terminal:
```
Ultrasonic Distance Monitor Online
Distance: 120 cm
Distance: 45 cm
```

## Expected Canvas Behavior
| HC-SR04 Slider Position | Calculated Distance |
| --- | --- |
| Close range | ~15–30 cm |
| Mid range | ~100–150 cm |
| Long range | > 200 cm |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pulseIn(ECHO_PIN, HIGH)` | Measures the duration (in microseconds) that `ECHO_PIN` remains in a `HIGH` state. |
| `duration * 0.0343 / 2` | Standard calculation formula converting travel time to distance based on the speed of sound. Divided by 2 because the signal travels to the target and back. |

## Hardware & Safety Concept: Echo Pin Voltage Dividers
The HC-SR04 operates at **5V** and outputs a 5V signal on its Echo pin. The Raspberry Pi Pico's GPIO pins are strictly **3.3V tolerant**. Connecting a 5V output directly to GP15 will damage the pin over time. To protect the Pico, use a **voltage divider** (e.g. 10k fixed resistor from Echo to GP15, and 20k resistor from GP15 to GND) to scale the 5V signal down to a safe 3.3V logic level.

## Try This! (Challenges)
1. **Out of Range Filter**: Modify the code to print "Out of Range" if the calculated distance is greater than 400 cm or equal to 0 cm.
2. **Crash Alert LED**: Turn ON an external LED on GP13 if the measured distance is less than 20 cm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Distance prints static `0` or maximum values | Echo pin connection loose | Check the voltage divider wiring. If the Echo pin logic never reaches the 3.3V logic threshold on GP15, `pulseIn` returns 0. |

## Mode Notes
This basic proximity sensor project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [76 - Pico Ultrasonic Alarm](76-pico-ultrasonic-alarm.md)
- [77 - Pico Ultrasonic LCD](77-pico-ultrasonic-lcd.md)
