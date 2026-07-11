# 47 - Joysticks Y-Axis Read (GPIO Analog pin)

Read and display the vertical Y-axis position of an analog joystick on the VEGA ARIES v3 board.

## Goal
Configure an analog pin to read the vertical potentiometer of a dual-axis joystick, interpret vertical directions (UP, CENTER, DOWN) using software threshold zones, and display them.

## What You Will Build
An analog joystick module's Y-axis output (`VRy`) is connected to analog input pin `ADC1` (`GP27`). Moving the joystick handle forward/up and backward/down changes the output voltage. The board reads this value and displays the raw ADC level and interpreted direction (`UP`, `CENTER`, or `DOWN`) on the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Joystick Module | `joystick` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Joystick Module | +5V (VCC) | 3.3V | Red | Power connection |
| Joystick Module | VRy (Y-Axis Output) | ADC1 (GP27) | Yellow | Analog Y-axis signal |
| Joystick Module | GND | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The Joystick module contains two independent potentiometers (for VRx and VRy). Connect the VRy pin to GP27 (ADC1). Leave VRx and the SW (Switch) pins disconnected for this project.

## Code
```cpp
// Joystick Y-Axis Read - VEGA ARIES v3
const int JOY_Y_PIN = GP27; // ADC1

void setup() {
  // Initialize serial communication at 115200 bps
  Serial.begin(115200);
}

void loop() {
  // Read the raw analog value of the Y-axis potentiometer
  int yVal = analogRead(JOY_Y_PIN);
  
  Serial.print("Joystick Y Raw: ");
  Serial.print(yVal);
  
  // Categorize position based on threshold values (including a center deadzone)
  // Up: values below 1500 | Down: values above 2500 | Center: in-between
  if (yVal < 1500) {
    Serial.println(" [UP]");
  } else if (yVal > 2500) {
    Serial.println(" [DOWN]");
  } else {
    Serial.println(" [CENTER]");
  }
  
  // Print update every 500 milliseconds
  delay(500);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and a **Joystick** onto the canvas.
2. Wire the Joystick's VRy pin to ARIES **GP27 (ADC1)**.
3. Connect VCC to **3.3V** and GND to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Click and drag the joystick handle vertically to observe the output in the Serial Monitor.

## Expected Output
Serial Monitor:
```
System Initialized.
Joystick Y Raw: 2048 [CENTER]
Joystick Y Raw: 450 [UP]
Joystick Y Raw: 3800 [DOWN]
```

## Expected Canvas Behavior
* The Serial Monitor output updates dynamically as you move the virtual joystick handle vertically on the canvas.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int JOY_Y_PIN = GP27;` | Defines the Y-axis input pin. |
| `analogRead(JOY_Y_PIN)` | Reads the raw 12-bit value (0-4095) from the Y-axis potentiometer. |
| `if (yVal < 1500)` | Threshold check: values below 1500 indicate that the joystick has been pushed UP. |
| `else if (yVal > 2500)` | Threshold check: values above 2500 indicate that the joystick has been pushed DOWN. |

## Hardware & Safety Concept: Dual Axis Potentiometers and Mechanical Limits
* **Dual Axis Potentiometers**: Inside the joystick assembly are two linear potentiometers mounted at 90-degree angles to each other. A mechanical gimbal converts the diagonal movements of the joystick shaft into proportional rotary sweeps of the two sliders.
* **Mechanical Limits**: Joysticks are limited to a small sweep angle (typically ~40 to 60 degrees of rotation) compared to standard panel-mount potentiometers (270 degrees). This results in a smaller dynamic voltage range in some joystick designs, requiring software tuning of the threshold levels.

## Try This! (Challenges)
1. **Pitch Indicator**: Connect an active buzzer to `GPIO 14`. Map the Y-axis values to control the beep frequency: sound the buzzer when pushed UP, with pitch increasing the further up it goes, and keep it silent at CENTER.
2. **Y-Axis Brightness Control**: Fade the onboard Green LED (`LED_G`) brightness using `analogWrite` proportional to Y-axis deflection.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reading is stuck at 0 or 4095 | Loose wiring on VRy, VCC, or GND | Verify the connection path from VRy to GP27 and ensure VCC and GND pins are securely seated |
| Y-axis behaves in reverse | Inverted wiring or directional conventions | If UP and DOWN are inverted in your setup, swap the comparison conditions (`< 1500` and `> 2500`) in your code or swap VCC and GND wires on the joystick |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - Joysticks X-Axis read](46-joysticks-x-axis-read.md) (Previous project)
- [48 - LDR ambient light LED dimmer](48-ldr-ambient-light-led-dimmer.md) (Next project)
