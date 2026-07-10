# 48 - Pico Joystick Y-Axis

Read and log the vertical (Y-axis) coordinates of an analog joystick.

## Goal
Learn how to interface dual-axis joysticks and capture vertical coordinate values using analog inputs on the Raspberry Pi Pico.

## What You Will Build
A joystick position reader:
- **Joystick Y-Axis (GP27)**: Connected to ADC1. Moving the joystick thumbstick up and down prints raw coordinate values (`0` to `4095`) to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Analog Joystick Module | `joystick` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Joystick | VCC | 3.3V | Power supply |
| Joystick | VRY (Y-Axis Out) | GP27 (ADC1) | Vertical analog output |
| Joystick | GND | GND | Ground reference |

## Code
```cpp
const int JOY_Y_PIN = 27; // GP27 (ADC1)

void setup() {
  Serial.begin(9600);
  pinMode(JOY_Y_PIN, INPUT);
  Serial.println("Joystick Y-Axis Monitor Active");
}

void loop() {
  int yVal = analogRead(JOY_Y_PIN); // Reads 0 to 4095

  Serial.print("Joystick Y Position: ");
  Serial.println(yVal);

  delay(200); // Sample 5 times per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Joystick** onto the canvas.
2. Connect Joystick: **VCC** to **3V3**, **VRY** to **GP27**, **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the joystick vertical coordinate handle and observe the Serial logs.

## Expected Output

Terminal:
```
Joystick Y-Axis Monitor Active
Joystick Y Position: 2048
Joystick Y Position: 0
```

## Expected Canvas Behavior
| Joystick Position | GP27 Read Value | Vertical Direction |
| --- | --- | --- |
| Fully Up | < 200 | Moving Up |
| Center (Idle) | 1900–2200 | Center / Deadzone |
| Fully Down | > 3900 | Moving Down |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `analogRead(JOY_Y_PIN)` | Reads the raw voltage representing the position of the vertical internal potentiometer. |

## Hardware & Safety Concept: Multi-Channel ADC Reads
The RP2040 chip can read multiple ADC channels sequentially. Reading VRX and VRY on separate pins allows coordinates mapping. However, because the ADC input multiplexer shares a single sample-and-hold capacitor inside the chip, high-speed switching between channels can cause small voltage "ghosting" effects. Adding a tiny delay between reads eliminates this crosstalk.

## Try This! (Challenges)
1. **Vertical Alert**: Sound a buzzer on GP14 if the stick is pushed fully up or down.
2. **Direction Logger**: Print "UP" if Y < 1000, "DOWN" if Y > 3000, and print nothing when centered.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| VRX and VRY show the same values | Swapped wires | Verify that VRX is wired to GP26 and VRY is wired to GP27 separately. |

## Mode Notes
This basic analog monitoring project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [47 - Pico Joystick X-Axis](47-pico-joystick-x.md)
- [97 - Pico Joystick Dual Servo](../../intermediate/97-pico-joystick-servo.md)
