# 47 - Pico Joystick X-Axis

Read and log the horizontal (X-axis) coordinates of an analog joystick.

## Goal
Learn how to interface dual-axis joysticks and capture coordinate values using analog inputs on the Raspberry Pi Pico.

## What You Will Build
A joystick position reader:
- **Joystick X-Axis (GP26)**: Connected to ADC0. Moving the joystick thumbstick left and right prints raw coordinate values (`0` to `4095`) to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Analog Joystick Module | `joystick` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Joystick | VCC | 3.3V | Power supply |
| Joystick | VRX (X-Axis Out) | GP26 (ADC0) | Horizontal analog output |
| Joystick | GND | GND | Ground reference |

## Code
```cpp
const int JOY_X_PIN = 26; // GP26 (ADC0)

void setup() {
  Serial.begin(9600);
  pinMode(JOY_X_PIN, INPUT);
  Serial.println("Joystick X-Axis Monitor Active");
}

void loop() {
  int xVal = analogRead(JOY_X_PIN); // Reads 0 to 4095

  Serial.print("Joystick X Position: ");
  Serial.println(xVal);

  delay(200); // Sample 5 times per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Joystick** onto the canvas.
2. Connect Joystick: **VCC** to **3V3**, **VRX** to **GP26**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the joystick horizontal coordinate handle and observe the Serial logs.

## Expected Output

Terminal:
```
Joystick X-Axis Monitor Active
Joystick X Position: 2048
Joystick X Position: 4095
```

## Expected Canvas Behavior
| Joystick Position | GP26 Read Value | Horizontal Direction |
| --- | --- | --- |
| Far Left | < 200 | Moving Left |
| Center (Idle) | 1900–2200 | Center / Deadzone |
| Far Right | > 3900 | Moving Right |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `analogRead(JOY_X_PIN)` | Reads the raw voltage representing the position of the horizontal internal potentiometer. |

## Hardware & Safety Concept: Joystick Potentiometers
Analog joysticks consist of two independent potentiometers mounted at 90-degree angles to measure X and Y axes. When idle, internal spring returns center the stick, outputting half of VCC (around 1.65V, reading roughly `2048` on a 12-bit ADC). Because mechanical tolerances vary, joysticks rarely read exactly `2048` at center, requiring a small **deadzone** filter in software to ignore minor drift.

## Try This! (Challenges)
1. **Direction LED**: Turn ON a green LED on GP15 when pushed fully right, and a red LED on GP14 when pushed fully left.
2. **Horizontal Deadzone**: Modify the code to print "LEFT" if X < 1000, "RIGHT" if X > 3000, and print nothing when centered.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Joystick readings drift continuously | Mechanical wear | Increase the software deadzone limits to account for spring centering tolerances. |

## Mode Notes
This basic analog monitoring project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [31 - Pico Potentiometer ADC](31-pico-potentiometer-adc.md)
- [48 - Pico Joystick Y-Axis](48-pico-joystick-y.md)
