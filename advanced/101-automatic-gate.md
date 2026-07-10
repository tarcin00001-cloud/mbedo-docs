# 101 - Automatic Gate

Use an IR obstacle sensor to detect an approaching vehicle or person and automatically open a servo-actuated gate, showing gate status on a 16x2 I2C LCD.

## Goal
Learn how to use a digital sensor to trigger a servo motor position change and update an LCD message — the fundamental pattern behind automated barriers, garage doors, and access gates.

## What You Will Build
When the IR sensor detects an obstacle (object within range) the servo sweeps from 0° (closed) to 90° (open) and the LCD shows "Gate: OPEN". When the obstacle clears the servo returns to 0° and the LCD shows "Gate: CLOSED". State changes are printed to the Serial Monitor.

**Why these pins?** The IR sensor outputs a digital LOW when an object is detected and goes to `D3`. The servo signal wire goes to `D6` (a PWM-capable pin required by the Servo library). The LCD communicates over the I2C bus on A4/A5.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| IR Obstacle Sensor | `ir_sensor` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| IR Sensor | VCC | 5V | Power supply |
| IR Sensor | OUT | D3 | Active-LOW digital output |
| IR Sensor | GND | GND | Ground reference |
| Servo Motor | VCC (Red) | 5V | Power supply |
| Servo Motor | Signal (Yellow) | D6 | PWM control |
| Servo Motor | GND (Brown) | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data |
| 16x2 I2C LCD | SCL | A5 | I2C Clock |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <Servo.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int IR_PIN    = 3;
const int SERVO_PIN = 6;

Servo gate;
LiquidCrystal_I2C lcd(0x27, 16, 2);

int lastState = -1; // Track previous state to avoid redundant updates

void setup() {
  Serial.begin(9600);
  pinMode(IR_PIN, INPUT);

  gate.attach(SERVO_PIN);
  gate.write(0); // Start closed

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Automatic Gate");
  lcd.setCursor(0, 1);
  lcd.print("Initialising...");
  delay(1500);
  lcd.clear();

  Serial.println("Automatic Gate Ready");
}

void loop() {
  int irState = digitalRead(IR_PIN);
  // IR sensor is active-LOW: LOW means object detected
  int detected = (irState == LOW) ? 1 : 0;

  if (detected != lastState) {
    if (detected == 1) {
      gate.write(90); // Open gate
      lcd.setCursor(0, 0);
      lcd.print("Gate: OPEN      ");
      lcd.setCursor(0, 1);
      lcd.print("Obstacle Detect.");
      Serial.println("Gate OPEN - obstacle detected");
    } else {
      gate.write(0);  // Close gate
      lcd.setCursor(0, 0);
      lcd.print("Gate: CLOSED    ");
      lcd.setCursor(0, 1);
      lcd.print("Path is clear   ");
      Serial.println("Gate CLOSED - path clear");
    }
    lastState = detected;
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **IR Obstacle Sensor**, **Servo Motor**, and **16x2 I2C LCD** onto the canvas.
2. Connect IR Sensor: **VCC** to **5V**, **OUT** to **D3**, **GND** to **GND**.
3. Connect Servo: **VCC** to **5V**, **Signal** to **D6**, **GND** to **GND**.
4. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.
8. Double-click the IR sensor on the canvas to toggle the obstacle state — observe the servo angle and LCD message change.

## Expected Output

Terminal:
```
Automatic Gate Ready
Gate OPEN - obstacle detected
Gate CLOSED - path clear
```

LCD Display:
```
Gate: OPEN
Obstacle Detect.
```

## Expected Canvas Behavior
| IR Sensor Output | Servo Angle | LCD Line 1 | LCD Line 2 |
| --- | --- | --- | --- |
| LOW (obstacle) | 90° | `Gate: OPEN` | `Obstacle Detect.` |
| HIGH (clear) | 0° | `Gate: CLOSED` | `Path is clear` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `irState == LOW` | Converts the active-LOW IR output to a logical `1` meaning "object present". |
| `detected != lastState` | Guards against running the servo and LCD update every 200 ms when nothing has changed. |
| `gate.write(90)` | Commands the servo to move to 90°, representing the open gate position. |
| `gate.write(0)` | Commands the servo to return to 0°, representing the closed gate position. |
| Trailing spaces `"      "` | Clears any residual characters on the LCD from longer previous strings. |

## Hardware & Safety Concept: Active-LOW Sensors
Many digital sensors, including most IR obstacle modules, output **active-LOW** signals — the output pin is pulled HIGH normally and goes LOW when the event (obstacle) is detected. This is the opposite of what intuition expects.
- Always check the sensor datasheet or silkscreen label for `OUT` polarity.
- Inverting the logic in code (`irState == LOW`) is cleaner than adding inverter hardware.
- Pull-up resistors (often internal to the sensor module) keep the line stable when no object is present.

## Try This! (Challenges)
1. **Entry Buzzer**: Add a buzzer to `D5` and call `tone(5, 1000, 300)` each time the gate opens, giving an audible entry chime.
2. **Hold-Open Delay**: Count loop iterations after the IR goes clear before closing, simulating a vehicle clearance delay so the gate does not close on a slow-moving vehicle.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Gate never opens | IR sensor polarity | Confirm the sensor is active-LOW; change condition to `irState == HIGH` if your module is active-HIGH. |
| Servo jitters constantly | No state-change guard | Ensure `lastState` logic is working so `gate.write()` only fires on transitions. |
| LCD stays blank | Wrong I2C address | Try changing `0x27` to `0x3F` in the constructor. |
| Gate opens but does not close | `lastState` not updating | Verify `lastState = detected` is inside the `if (detected != lastState)` block. |

## Mode Notes
All patterns used here (`digitalRead`, `Servo.write`, `lcd.setCursor`, `lcd.print`, `if/else`) are fully supported by MbedO interpreted mode.

## Related Projects
- [100 - Smart Fan](100-smart-fan.md)
- [102 - BT Home Controller](102-bt-home-controller.md)
- [57 - PIR Motion Alert](../intermediate/57-pir-motion-alert.md)
- [90 - Servo Sweep](90-servo-sweep.md)
