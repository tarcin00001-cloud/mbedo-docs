# 80 - Distance Display TM1637

Display real-time distance measurements from an HC-SR04 Ultrasonic Sensor on a 4-digit TM1637 7-segment display.

## Goal
Learn how to read an ultrasonic distance sensor (HC-SR04) and display the resulting range values in centimeters on a 7-segment display.

## What You Will Build
The Arduino measures the distance to the nearest object using the ultrasonic sensor and displays the value (in centimeters, e.g. `120`) on the TM1637 display screen, updating every 200 ms.

**Why D2, D3, D4, and D5?** Pins D2/D3 manage the display. Pins D4/D5 trigger and read the ultrasonic sensor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-SR04 Ultrasonic | `hc_sr04` | Yes | Yes |
| TM1637 Display | `seven_segment_tm1637` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 Sensor | VCC | 5V | Power supply |
| HC-SR04 Sensor | TRIG | D4 | Trigger connection |
| HC-SR04 Sensor | ECHO | D5 | Echo connection |
| HC-SR04 Sensor | GND | GND | Ground reference |
| TM1637 Display | CLK | D2 | Serial Clock line |
| TM1637 Display | DIO | D3 | Serial Data line |
| TM1637 Display | VCC | 5V | Power supply (5V) |
| TM1637 Display | GND | GND | Ground reference |

## Code
```cpp
#include <TM1637Display.h>

const int CLK_PIN  = 2;
const int DIO_PIN  = 3;
const int TRIG_PIN = 4;
const int ECHO_PIN = 5;

TM1637Display display(CLK_PIN, DIO_PIN);

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  display.setBrightness(5);
  display.clear();
  
  Serial.begin(9600);
  Serial.println("Ultrasonic TM1637 Station Active");
}

void loop() {
  // Trigger ultrasonic reading
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH);
  long distance = duration * 0.034 / 2;
  
  Serial.print("Measured Distance: ");
  Serial.print(distance);
  Serial.println(" cm");
  
  // Display the distance on the 7-segment display
  if (distance > 0 && distance < 400) {
    display.showNumberDec(distance);
  } else {
    display.clear(); // Clear display if out of range
  }
  
  delay(200); // 5 Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-SR04 Sensor**, and **TM1637 7-Segment Display** onto the canvas.
2. Connect HC-SR04: **VCC** to **5V**, **TRIG** to **D4**, **ECHO** to **D5**, and **GND** to **GND**.
3. Connect TM1637: **CLK** to **D2**, **DIO** to **D3**, **VCC** to **5V**, and **GND** to **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Double-click the HC-SR04 sensor, adjust its distance slider, and watch the values display on the 7-segment screen.

## Expected Output
The TM1637 display component on the canvas lights up and displays:
```
 120
```
The number shown tracks the sensor distance slider value.

## Expected Canvas Behavior

| Distance Slider Input | Calculated Range | TM1637 Display Output |
| --- | --- | --- |
| 55 cm | 55 cm | `  55` |
| 180 cm | 180 cm | ` 180` |
| 450 cm (Out of range) | > 400 cm | (Blank Display) |

The display updates automatically every 200 ms.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `display.showNumberDec(distance)` | Draws the calculated distance number directly to the 7-segment LEDs. |

## Hardware & Safety Concept: Numeric Display Formatting
A 4-digit display can show numbers from `0` to `9999`. 
- By default, `showNumberDec()` aligns numbers to the right side of the screen.
- If a reading drops to single digits (e.g. 5 cm), it displays as `   5` (preceded by spaces).
- You can specify leading zeros in the library (e.g., `display.showNumberDec(5, true)` to show `0005`), which is useful for formatting clock digits or coordinate locations.

## Try This! (Challenges)
1. **Show Inches**: Change the conversion logic to calculate and display the range in inches.
2. **Close Proximity Alarm**: If the distance drops below 10 cm, display a flashing `9999` to grab attention.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Display goes completely blank | Sensor returning 0 or out of bounds | Check the Terminal to confirm the raw sensor reading is valid. Verify that the echo pin matches D5. |
| Numbers display garbled | Segment wiring swapped | Check that CLK connects to D2 and DIO connects to D3. |

## Mode Notes
These patterns (ultrasonic reads combined with TM1637 display printing) are supported by MbedO interpreted mode.

## Related Projects
- [54 - Distance Serial](54-distance-serial.md)
- [57 - Distance Display LCD](57-distance-display-lcd.md)
- [78 - Potentiometer Number](78-potentiometer-number.md)
