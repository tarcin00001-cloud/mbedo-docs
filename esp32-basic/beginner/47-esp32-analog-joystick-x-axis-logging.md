# 47 - ESP32 Analog Joystick X-Axis Logging

Read the X-axis analog output of a dual-axis joystick module and log its position as a raw ADC value, a voltage, and a directional label.

## Goal
Learn how a joystick module outputs two independent analog voltages for X and Y axes, how to read the X-axis on the ESP32 ADC, and how to classify the stick position into directional zones.

## What You Will Build
A joystick module X-axis output connected to GPIO 34. The code reads the ADC at rest (~2048 centre), left (~0), and right (~4095), and prints the value alongside a directional label (LEFT / CENTRE / RIGHT).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Analog Dual-Axis Joystick Module | `joystick` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Joystick Module | VCC (+5V) | 3V3 | Red | Module supply (3V3 also works for this module) |
| Joystick Module | GND | GND | Black | Module ground |
| Joystick Module | VRx (X-axis) | GPIO34 | Yellow | X-axis analog output |
| Joystick Module | VRy (Y-axis) | Not connected | — | Y-axis unused in this project |
| Joystick Module | SW (button) | Not connected | — | Push-button unused in this project |

> **Wiring tip:** Joystick modules are simply two potentiometers mechanically linked to a pivoting stick. VRx outputs a voltage proportional to horizontal displacement; VRy to vertical displacement. The centre resting position outputs approximately half the supply voltage (≈1.65 V with 3V3 supply → ADC ≈ 2048). Tilting left drives VRx toward 0 V; tilting right drives it toward 3V3. GPIO 34 is an input-only ADC pin, ideal for this passive analog signal.

## Code
```cpp
// Analog Joystick X-Axis Logger
const int JOY_X_PIN  = 34;

// Dead-zone thresholds around the centre resting position
const int CENTRE_LOW  = 1800;   // Below centre dead-zone
const int CENTRE_HIGH = 2300;   // Above centre dead-zone

String classifyX(int raw) {
  if      (raw < CENTRE_LOW)  return "LEFT";
  else if (raw > CENTRE_HIGH) return "RIGHT";
  else                        return "CENTRE";
}

void setup() {
  Serial.begin(115200);
  Serial.println("Joystick X-Axis Logger ready.");
}

void loop() {
  int raw     = analogRead(JOY_X_PIN);
  float volts = raw * (3.3f / 4095.0f);

  Serial.print("X ADC: "); Serial.print(raw);
  Serial.print("  |  "); Serial.print(volts, 2); Serial.print(" V");
  Serial.print("  |  Direction: ");
  Serial.println(classifyX(raw));

  delay(200);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Joystick** onto the canvas.
2. Connect Joystick **VRx** to **GPIO34**.
3. Paste the code and click **Run**.
4. Drag the joystick widget left and right — observe directional labels.

## Expected Output
Serial Monitor:
```
Joystick X-Axis Logger ready.
X ADC: 2048  |  1.65 V  |  Direction: CENTRE
X ADC: 120   |  0.10 V  |  Direction: LEFT
X ADC: 3980  |  3.21 V  |  Direction: RIGHT
X ADC: 2100  |  1.69 V  |  Direction: CENTRE
```

## Expected Canvas Behavior
* Centre position logs "CENTRE" with ADC near 2048.
* Moving the joystick left logs "LEFT" and decreasing ADC values.
* Moving the joystick right logs "RIGHT" and increasing ADC values.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `const int CENTRE_LOW = 1800` | Lower bound of the centre dead-zone — values below this classify as LEFT. |
| `const int CENTRE_HIGH = 2300` | Upper bound of the centre dead-zone — values above this classify as RIGHT. |
| `classifyX(raw)` | Returns a direction string based on which zone the reading falls in. |
| `delay(200)` | 5 Hz update rate — fast enough for manual joystick control, readable in Serial Monitor. |

## Hardware & Safety Concept: Dead-Zone Filtering in Joystick Control
A physical joystick at rest does not output exactly half the supply voltage — mechanical tolerances, friction, and component variation cause the resting position to drift by ±100–300 ADC counts from the ideal centre (2048). Without a **dead zone**, a joystick left alone would continuously report random LEFT/RIGHT commands due to this resting noise. A dead zone defines a range around centre that is treated as "no input". Any reading within the dead zone is classified as CENTRE regardless of the exact value. Dead zones are used in: game controller firmware (the PS4/Xbox controller has a built-in dead zone), industrial crane joysticks (prevents unintended load movement), drone flight controllers (prevents drift when hovering), and servo pan-tilt systems.

## Try This! (Challenges)
1. **Y-axis**: Connect VRy to GPIO 35 and log both axes simultaneously as (X_dir, Y_dir).
2. **LED direction indicator**: Light LED A when left, LED B when right, both off when centre.
3. **Speed mapping**: Map the X-axis raw value to a motor speed (0–255 PWM) where centre = stop and either extreme = full speed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Always reads LEFT or RIGHT | Joystick VRx not centred or resting position off | Adjust `CENTRE_LOW` and `CENTRE_HIGH` to bracket your actual resting ADC value |
| ADC reads constant 0 | VRx wire loose or missing | Re-seat the GPIO 34 wire in VRx pin |
| Readings skip between extremes with no movement | Noise on long VRx wire | Shorten wire; add 100 nF capacitor from GPIO 34 to GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [31 - ESP32 Potentiometer ADC Read](31-esp32-potentiometer-adc-read.md)
- [48 - ESP32 Analog Joystick Y-Axis Logging](48-esp32-analog-joystick-y-axis-logging.md)
- [46 - ESP32 Potentiometer Servo Control](46-esp32-potentiometer-servo-control.md)
