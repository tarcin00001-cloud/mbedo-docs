# 37 - Pico IR Obstacle

Detect nearby obstacles using an Infrared (IR) Proximity Sensor.

## Goal
Learn how to interface active infrared transceiver modules with digital GPIO input pins on the Raspberry Pi Pico.

## What You Will Build
An obstacle detection warning system:
- **IR Obstacle Sensor (GP16)**: Reads `LOW` when an object is placed in front of the sensor (beam reflected), and `HIGH` when the path is clear.
- **Warning LED (GP15)**: Turns ON (Red alert) immediately when an obstacle is detected.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| IR Obstacle Sensor | `button` | Yes (represented by switch button) | Yes (or IR sensor module) |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| IR Sensor | VCC | 3.3V | Power supply |
| IR Sensor | OUT | GP16 | Digital signal output |
| IR Sensor | GND | GND | Ground reference |
| Red LED | Anode | GP15 | Obstacle alert indicator |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int IR_PIN  = 16;
const int LED_PIN = 15;

void setup() {
  pinMode(IR_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int obstacleDetected = digitalRead(IR_PIN);

  // Active-LOW logic: OUT is pulled to GND when beam is reflected back
  if (obstacleDetected == LOW) {
    digitalWrite(LED_PIN, HIGH); // Obstacle near! LED ON
  } else {
    digitalWrite(LED_PIN, LOW);  // Path clear, LED OFF
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **IR Obstacle Sensor** (represented by switch button), and **Red LED** onto the canvas.
2. Connect IR Sensor: **VCC** to **3.3V**, **OUT** to **GP16**, **GND** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Simulate an obstacle by closing the switch contact on canvas.

## Expected Output

Terminal:
```
Simulation active. GP16 IR obstacle sensor online.
```

## Expected Canvas Behavior
| IR Obstacle State | GP16 Input State | LED state (GP15) | Path Status |
| --- | --- | --- | --- |
| No object (Clear)| HIGH | LOW | Path Clear |
| Object near (Blocked)| LOW | HIGH | **OBSTACLE DETECTED** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `obstacleDetected == LOW` | Checks if the infrared receiver diode has turned ON from reflected IR light, bringing OUT to GND. |

## Hardware & Safety Concept: IR Proximity Sensing
IR obstacle modules contain an infrared LED transmitter and a photodiode receiver. The transmitter emits IR light. If an object is in range, the light bounces off the object back into the receiver diode. A built-in comparator (like LM393) checks this input voltage against a sensitivity potentiometer and outputs a clean digital `LOW` signal if the threshold is met.

## Try This! (Challenges)
1. **Collision Alarm**: Add a buzzer on GP14 and sound a rapid warning chirp when an obstacle is close.
2. **Reverse Indicator**: Turn the LED ON when the path is clear, and turn it OFF when blocked.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor registers false reflections | Sunlight interference | Direct sunlight contains high amounts of infrared light which saturate the receiver. Test in indoor environments. |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [24 - Pico Limit Switch](24-pico-limit-switch.md)
- [36 - Pico PIR Motion](36-pico-pir-motion.md)
