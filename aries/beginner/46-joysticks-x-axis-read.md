# 46 - Joysticks X-Axis Read (GPIO Analog pin)

Read and display the horizontal X-axis position of an analog joystick on the VEGA ARIES v3 board.

## Goal
Understand the electrical operation of dual-axis analog joysticks, configure an analog pin to read the X-axis potentiometer, and implement directional threshold zones (deadzones) in code.

## What You Will Build
An analog joystick module's X-axis output (`VRx`) is connected to analog input pin `ADC0` (`GP26`). Moving the joystick left and right changes the voltage. The board reads this voltage and prints the raw value along with the interpreted direction (`LEFT`, `CENTER`, or `RIGHT`) to the Serial Monitor.

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
| Joystick Module | VRx (X-Axis Output) | ADC0 (GP26) | Yellow | Analog X-axis signal |
| Joystick Module | GND | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The Joystick module contains two independent potentiometers (for VRx and VRy). Connect the VRx pin to GP26 (ADC0). Leave VRy and the SW (Switch) pins disconnected for this project.

## Code
```cpp
// Joystick X-Axis Read - VEGA ARIES v3
const int JOY_X_PIN = GP26; // ADC0

void setup() {
  // Initialize serial communication at 115200 bps
  Serial.begin(115200);
}

void loop() {
  // Read the raw analog value of the X-axis potentiometer
  int xVal = analogRead(JOY_X_PIN);
  
  Serial.print("Joystick X Raw: ");
  Serial.print(xVal);
  
  // Categorize position based on threshold values (including a center deadzone)
  // Left: values below 1500 | Right: values above 2500 | Center: in-between
  if (xVal < 1500) {
    Serial.println(" [LEFT]");
  } else if (xVal > 2500) {
    Serial.println(" [RIGHT]");
  } else {
    Serial.println(" [CENTER]");
  }
  
  // Print update every 500 milliseconds
  delay(500);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and a **Joystick** onto the canvas.
2. Wire the Joystick's VRx pin to ARIES **GP26 (ADC0)**.
3. Connect VCC to **3.3V** and GND to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Click and drag the joystick handle horizontally to observe the output in the Serial Monitor.

## Expected Output
Serial Monitor:
```
System Initialized.
Joystick X Raw: 2048 [CENTER]
Joystick X Raw: 450 [LEFT]
Joystick X Raw: 3800 [RIGHT]
```

## Expected Canvas Behavior
* The Serial Monitor output updates dynamically as you move the virtual joystick handle left and right on the canvas.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int JOY_X_PIN = GP26;` | Defines the X-axis input pin. |
| `analogRead(JOY_X_PIN)` | Reads the raw 12-bit value (0-4095) from the potentiometer. |
| `if (xVal < 1500)` | Threshold check: values below 1500 indicate that the joystick has been pushed to the left. |
| `else if (xVal > 2500)` | Threshold check: values above 2500 indicate that the joystick has been pushed to the right. |

## Hardware & Safety Concept: Joystick Center Baselines and Deadzones
* **Center Baselines**: Dual-axis joysticks are spring-loaded to return to the center. Each axis behaves as a voltage divider. At rest, the output sits at roughly VCC/2 (approx 2048).
* **Deadzones**: Due to mechanical tolerances and spring wear, joysticks rarely return to *exactly* 2048. To prevent false movements or "drift" in games or robot controls, software must implement a deadzone (e.g. ignoring all changes between 1800 and 2300).

## Try This! (Challenges)
1. **LED Direction Indicator**: Turn ON the onboard Red LED (`LED_R`) when the joystick is pushed LEFT and the onboard Blue LED (`LED_B`) when it is pushed RIGHT. Keep both OFF when CENTER.
2. **Visual Value Mapping**: Map the X-axis value (0-4095) directly to the brightness (0-255 PWM duty cycle) of an external LED on `GPIO 15` so that pushing the joystick to the right brightens the LED, and pushing to the left dims it.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reading is stuck at 0 or 4095 | Loose wiring on VRx, VCC, or GND | Verify the connection path from VRx to GP26 and ensure VCC and GND pins are securely seated |
| Values jump erratically when stationary | Floating ground or bad connection | Ensure the Joystick GND is connected to the ARIES board's GND to create a stable voltage reference |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [45 - Potentiometer servo sweep](45-potentiometer-servo-sweep.md) (Previous project)
- [47 - Joysticks Y-Axis read](47-joysticks-y-axis-read.md) (Next project)
