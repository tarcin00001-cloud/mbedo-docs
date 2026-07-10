# 37 - ESP32 IR Obstacle Sensor Print

Use an IR obstacle detection module to detect objects in front of the sensor and print the detection state to the Serial Monitor.

## Goal
Learn how an IR obstacle sensor module works, how to read its active-LOW digital output, and how to print descriptive detection states using `digitalRead()`.

## What You Will Build
An IR obstacle sensor module on GPIO 4. When an obstacle is detected within range, the module pulls DO LOW. The ESP32 reads this and prints "OBSTACLE DETECTED" to the Serial Monitor. Clear space prints "Path clear."

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| IR Obstacle Sensor Module (FC-51) | `ir_sensor` | Yes | Yes |
| LED (red) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Sensor Module | VCC | 3V3 | Red | Module supply voltage |
| IR Sensor Module | GND | GND | Black | Module ground |
| IR Sensor Module | DO (OUT) | GPIO4 | Yellow | Active-LOW output — LOW on detection |
| LED (red) | Anode (+) | GPIO5 via 330 Ω | Orange | Obstacle indicator |
| LED (red) | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** The FC-51 IR obstacle module has a sensitivity trimmer. Rotate it clockwise to increase detection range (up to ~30 cm) and counter-clockwise to decrease it. The module's DO pin uses active-LOW logic: DO = LOW when an obstacle is detected, HIGH when clear. Keep the sensor away from reflective surfaces and strong ambient IR sources (sunlight, incandescent bulbs) which can cause false readings.

## Code
```cpp
// IR Obstacle Sensor — active-LOW digital output
const int IR_PIN  = 4;
const int LED_PIN = 5;

void setup() {
  pinMode(IR_PIN,  INPUT);      // Module drives pin actively — no pull needed
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Serial.begin(115200);
  Serial.println("IR Obstacle Sensor ready.");
}

void loop() {
  int irState = digitalRead(IR_PIN);

  // Active-LOW: LOW = obstacle detected, HIGH = path clear
  if (irState == LOW) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("!! OBSTACLE DETECTED !!");
  } else {
    digitalWrite(LED_PIN, LOW);
    Serial.println("Path clear.");
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **IR Sensor**, and **LED** onto the canvas.
2. Connect IR Sensor **DO** to **GPIO4**, LED **anode** to **GPIO5**.
3. Paste the code and click **Run**.
4. Toggle the IR sensor widget to simulate an obstacle.

## Expected Output
Serial Monitor:
```
IR Obstacle Sensor ready.
Path clear.
Path clear.
!! OBSTACLE DETECTED !!
!! OBSTACLE DETECTED !!
Path clear.
```

## Expected Canvas Behavior
* LED is off while the sensor widget shows clear.
* Activating the IR sensor widget immediately lights the LED.
* LED extinguishes when the widget is deactivated.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `int irState = digitalRead(IR_PIN)` | Reads the module's digital output — LOW on obstacle, HIGH on clear. |
| `if (irState == LOW)` | Active-LOW detection: obstacle present drives DO to GND (LOW). |
| `digitalWrite(LED_PIN, HIGH)` | Lights the indicator LED on obstacle detection. |
| `delay(200)` | 5 Hz polling rate — fast enough to detect a moving object. |

## Hardware & Safety Concept: IR Reflection-Based Obstacle Detection
The FC-51 module emits infrared light from an IR LED and monitors reflected IR with a photodiode. Objects with a reflective surface reflect the emitted IR back to the receiver. The module's comparator circuit triggers DO LOW when the reflected signal exceeds a threshold. Detection range and sensitivity depend on the object's **IR reflectivity**: white paper reflects well and is easily detected at 30 cm; black fabric absorbs IR and may only trigger at 2–5 cm. Smooth, shiny surfaces (mirrors, polished metal) can reflect IR at an angle away from the receiver and be missed entirely. This technology is used in obstacle avoidance robots, printers (paper detection), conveyor belt sensors, and proximity-based automatic hand sanitizer dispensers.

## Try This! (Challenges)
1. **Object counter**: Count each time an obstacle enters the detection zone (rising edge of DO going LOW) and log the total.
2. **Range indicator**: Add two LEDs on different GPIO pins — one for "close" (under 5 cm, sensitivity high) and one for "medium" distance.
3. **Robot integration preview**: Add a buzzer on GPIO 15 that beeps when the obstacle is detected — simulating a collision warning.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor always reads LOW (always detected) | Sensitivity too high — detecting background | Rotate sensitivity trimmer counter-clockwise to reduce range |
| Sensor never triggers | Object not reflective enough, or range too short | Increase sensitivity; test with a white piece of paper |
| LED and logic inverted | Confusion about active-LOW polarity | Remember: LOW = obstacle, HIGH = clear for DO on the FC-51 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [36 - ESP32 PIR Motion Sensor Alert](36-esp32-pir-motion-sensor-alert.md)
- [38 - ESP32 Piezo Vibration Analog Signal Reader](38-esp32-piezo-vibration-analog-signal-reader.md)
- [30 - ESP32 Laser Receiver Signal Detector](30-esp32-laser-receiver-signal-detector.md)
