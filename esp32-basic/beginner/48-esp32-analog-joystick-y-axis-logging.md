# 48 - ESP32 Analog Joystick Y-Axis Logging

Read the Y-axis analog output of a dual-axis joystick module and log its position as a raw ADC value, voltage, and directional label (UP / CENTRE / DOWN).

## Goal
Learn how the Y-axis of a joystick module produces an independent analog voltage, read it on a second ADC pin, and combine X and Y axis readings into a full two-axis log.

## What You Will Build
A joystick module Y-axis output on GPIO 35. The code reads the ADC, classifies the stick as UP, CENTRE, or DOWN with a dead-zone, and prints the label with the raw value every 200 ms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Analog Dual-Axis Joystick Module | `joystick` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Joystick Module | VCC (+5V) | 3V3 | Red | Module supply voltage |
| Joystick Module | GND | GND | Black | Module ground |
| Joystick Module | VRx (X-axis) | GPIO34 | Yellow | X-axis (optional, logged alongside Y) |
| Joystick Module | VRy (Y-axis) | GPIO35 | Green | Y-axis analog output — primary input |
| Joystick Module | SW (button) | Not connected | — | Unused in this project |

> **Wiring tip:** GPIO 35 is another input-only ADC1 pin on the ESP32 — perfect for a second analog channel alongside GPIO 34. When both axes are wired simultaneously, you have a complete two-axis joystick reader. Note that the Y-axis direction convention may be inverted depending on the module orientation — if UP reports high ADC values, swap the label in the `classifyY()` function.

## Code
```cpp
// Analog Joystick Y-Axis Logger (with optional X-axis)
const int JOY_X_PIN  = 34;   // X-axis (optional, wired for reference)
const int JOY_Y_PIN  = 35;   // Y-axis — primary axis in this project

// Dead-zone thresholds
const int CENTRE_LOW  = 1800;
const int CENTRE_HIGH = 2300;

String classifyY(int raw) {
  // On most modules: low ADC = DOWN (forward push), high ADC = UP (backward pull)
  if      (raw < CENTRE_LOW)  return "DOWN";
  else if (raw > CENTRE_HIGH) return "UP";
  else                        return "CENTRE";
}

String classifyX(int raw) {
  if      (raw < CENTRE_LOW)  return "LEFT";
  else if (raw > CENTRE_HIGH) return "RIGHT";
  else                        return "CENTRE";
}

void setup() {
  Serial.begin(115200);
  Serial.println("Joystick Y-Axis Logger ready (X+Y logged).");
}

void loop() {
  int rawX = analogRead(JOY_X_PIN);
  int rawY = analogRead(JOY_Y_PIN);

  Serial.print("X: "); Serial.print(rawX);
  Serial.print(" ["); Serial.print(classifyX(rawX)); Serial.print("]");
  Serial.print("   Y: "); Serial.print(rawY);
  Serial.print(" ["); Serial.print(classifyY(rawY)); Serial.println("]");

  delay(200);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Joystick** onto the canvas.
2. Connect Joystick **VRx** to **GPIO34**, **VRy** to **GPIO35**.
3. Paste the code and click **Run**.
4. Move the joystick widget in all four directions — observe X and Y labels change together.

## Expected Output
Serial Monitor:
```
Joystick Y-Axis Logger ready (X+Y logged).
X: 2048 [CENTRE]   Y: 2048 [CENTRE]
X: 120  [LEFT]     Y: 2060 [CENTRE]
X: 2040 [CENTRE]   Y: 3990 [UP]
X: 3980 [RIGHT]    Y: 120  [DOWN]
X: 2050 [CENTRE]   Y: 2040 [CENTRE]
```

## Expected Canvas Behavior
* Centre resting position shows CENTRE for both axes.
* Pushing the joystick widget up logs Y: UP; down logs Y: DOWN.
* Combined diagonal movement shows both X and Y labels simultaneously.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `int rawX = analogRead(JOY_X_PIN)` | Reads the X-axis ADC on GPIO 34. |
| `int rawY = analogRead(JOY_Y_PIN)` | Reads the Y-axis ADC on GPIO 35. |
| `classifyY(rawY)` | Returns UP, DOWN, or CENTRE based on dead-zone thresholds. |
| `classifyX(rawX)` | Returns LEFT, RIGHT, or CENTRE for the horizontal axis. |
| Combined print | Formats both axes on a single line for easy reading. |

## Hardware & Safety Concept: Two-Axis Joystick Signal Architecture
A dual-axis joystick module contains two independent 10 kΩ potentiometers. The X-axis pot wiper moves left–right with the stick's horizontal displacement; the Y-axis pot wiper moves up–down with vertical displacement. Each axis produces an independent analog voltage completely decoupled from the other. Sampling both ADC channels in sequence (with a few microseconds between reads) provides a snapshot of the joystick position. For real-time robotic control or gaming applications, the sampling rate matters — at 200 ms, fast stick movements will be missed. For smoother control, reduce the `delay` to 20–50 ms. Two-axis joystick inputs are standard in: RC transmitters, gamepad controllers, industrial crane controls, surgical robot hand controllers, and camera gimbal remote systems.

## Try This! (Challenges)
1. **8-direction mapper**: Combine X and Y classifiers to produce 8 directions (N, NE, E, SE, S, SW, W, NW, CENTRE).
2. **Push button**: Connect SW to GPIO 16 with `INPUT_PULLUP` and detect joystick click (pressing the stick down).
3. **Servo pan-tilt**: Map X-axis to a horizontal servo (GPIO 5) and Y-axis to a vertical servo (GPIO 15) for a camera gimbal.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Y-axis always reads CENTRE | VRy wire not connected to GPIO 35 | Re-seat the green VRy wire |
| Y UP/DOWN seems swapped | Module mounted or labelled with inverted convention | Swap the return strings in `classifyY()` |
| Both axes jitter at rest | Insufficient dead-zone | Widen `CENTRE_LOW` to 1600 and `CENTRE_HIGH` to 2500 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [47 - ESP32 Analog Joystick X-Axis Logging](47-esp32-analog-joystick-x-axis-logging.md)
- [46 - ESP32 Potentiometer Servo Control](46-esp32-potentiometer-servo-control.md)
- [31 - ESP32 Potentiometer ADC Read](31-esp32-potentiometer-adc-read.md)
