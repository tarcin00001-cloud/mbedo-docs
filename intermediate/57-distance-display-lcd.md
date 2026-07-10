# 57 - Distance Display LCD

Display real-time distance measurements from an HC-SR04 Ultrasonic Sensor on a 16x2 I2C LCD screen.

## Goal
Learn how to read an ultrasonic distance sensor, format the output in centimeters, and display the result on an I2C character LCD screen.

## What You Will Build
The Arduino triggers the ultrasonic sensor, measures the distance, clears the LCD screen, and displays:
- Line 1: "Distance Monitor"
- Line 2: "Range: 120 cm"
The display refreshes every 250 ms.

**Why D2, D3, A4, and A5?** Pins D2/D3 measure the sensor. Pins A4 (SDA) and A5 (SCL) comprise the I2C bus controlling the LCD screen.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-SR04 Ultrasonic | `hc_sr04` | Yes | Yes |
| LCD I2C 16x2 | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 Sensor | VCC | 5V | Power supply (5V) |
| HC-SR04 Sensor | TRIG | D2 | Trigger connection |
| HC-SR04 Sensor | ECHO | D3 | Echo connection |
| HC-SR04 Sensor | GND | GND | Ground reference |
| LCD Module | VCC | 5V | Power supply (5V) |
| LCD Module | SDA | A4 | I2C Serial Data line |
| LCD Module | SCL | A5 | I2C Serial Clock line |
| LCD Module | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int TRIG_PIN = 2;
const int ECHO_PIN = 3;

LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  // Initialize the LCD display
  lcd.init();
  lcd.backlight();
  lcd.clear();
  
  // Print startup header
  lcd.setCursor(0, 0);
  lcd.print("Distance Monitor");
  lcd.setCursor(0, 1);
  lcd.print("Sensor active...");
  delay(1500);
}

void loop() {
  // Trigger reading
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH);
  long distance = duration * 0.034 / 2;
  
  lcd.clear();
  
  // Row 0: Header
  lcd.setCursor(0, 0);
  lcd.print("Distance Monitor");
  
  // Row 1: Output
  lcd.setCursor(0, 1);
  if (distance > 0 && distance < 400) {
    lcd.print("Range: ");
    lcd.print(distance);
    lcd.print(" cm");
  } else {
    lcd.print("Range: Out of bounds");
  }
  
  delay(250); // Refresh rate
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-SR04 Sensor**, and **LCD I2C 16x2** onto the canvas.
2. Connect HC-SR04 **VCC** to Arduino **5V**, **TRIG** to Arduino **D2**, **ECHO** to Arduino **D3**, and **GND** to Arduino **GND**.
3. Connect LCD **VCC** to Arduino **5V**, **GND** to Arduino **GND**, **SDA** to Arduino **A4**, and **SCL** to Arduino **A5**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Double-click the HC-SR04 sensor, adjust the distance slider, and watch the values display on the LCD screen on the canvas.

## Expected Output
The LCD screen component on the canvas lights up and displays:
```
Distance Monitor
Range: 120 cm
```
As you move the slider on the HC-SR04 sensor, the display updates to reflect the new values.

## Expected Canvas Behavior

| Distance Slider | LCD Line 1 Display | LCD Line 2 Display |
| --- | --- | --- |
| 150 cm | "Distance Monitor" | "Range: 150 cm" |
| 450 cm (Max) | "Distance Monitor" | "Range: Out of bounds" |

The display updates automatically every 250 ms.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `#include <LiquidCrystal_I2C.h>` | Imports the I2C LCD control library. |
| `lcd.print(distance)` | Writes the calculated distance number directly to the LCD cursor position. |

## Hardware & Safety Concept: Sensor Refresh Limits
The speed of sound travels through air at roughly 340 meters per second. A distance reading takes up to 25 milliseconds to complete (if the target is 4 meters away). If your code triggers the sensor too quickly (e.g. without any delay), the outgoing sound pulses can interfere with previous echo reflections, causing false readings. A delay of at least 60 ms is recommended between consecutive triggers.

## Try This! (Challenges)
1. **Inch Switch**: Add a second button to D4. If pressed, the display converts and displays the distance in inches instead of centimeters.
2. **Visual Warning Meter**: If the distance drops below 30 cm, write a warning "CLOSE TARGET!" on row 0 instead of the header.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD prints "Range: Out of bounds" constantly | Pin connection failure | Ensure TRIG connects to D2 and ECHO connects to D3. |
| LCD is blank but backlight is lit | Address or wiring error | Verify SDA goes to A4 and SCL goes to A5. Check address is `0x27`. |

## Mode Notes
These patterns (ultrasonic measurements combined with I2C LCD printing) are supported by MbedO interpreted mode.

## Related Projects
- [51 - Temp Display LCD](51-temp-display-lcd.md)
- [54 - Distance Serial](54-distance-serial.md)
- [55 - Distance Alert LED](55-distance-alert-led.md)
