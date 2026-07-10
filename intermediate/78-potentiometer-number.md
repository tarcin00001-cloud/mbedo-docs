# 78 - Potentiometer Number

Display the raw analog reading (0 to 1023) from a potentiometer dial on a 4-digit TM1637 7-segment display.

## Goal
Learn how to initialize and write numeric values to a TM1637 display module using the `TM1637Display.h` library.

## What You Will Build
As you turn the potentiometer dial on the canvas, the TM1637 display shows the current raw analog read value (e.g. `512` at center, `1023` at max) in real-time, updating every 100 ms.

**Why A0, D2, and D3?** Pin A0 measures the analog dial voltage. Pins D2 (CLK) and D3 (DIO) comprise the serial clock and data lines used to control the display.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| TM1637 Display | `seven_segment_tm1637` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Potentiometer | 1 | 5V | Positive reference |
| Potentiometer | 2 (Wiper) | A0 | Analog signal connection |
| Potentiometer | 3 | GND | Ground reference |
| TM1637 Display | CLK | D2 | Serial Clock line |
| TM1637 Display | DIO | D3 | Serial Data Input Output line |
| TM1637 Display | VCC | 5V | Power supply (5V) |
| TM1637 Display | GND | GND | Ground reference |

## Code
```cpp
#include <TM1637Display.h>

const int CLK_PIN = 2;
const int DIO_PIN = 3;
const int POT_PIN = A0;

// Initialize the display object named 'display'
TM1637Display display(CLK_PIN, DIO_PIN);

void setup() {
  Serial.begin(9600);
  
  // Set display brightness (0 is lowest, 7 is highest)
  display.setBrightness(5);
  display.clear();
}

void loop() {
  int sensorVal = analogRead(POT_PIN);
  
  Serial.print("Pot value: ");
  Serial.println(sensorVal);
  
  // Display the number on the 4-digit display screen
  display.showNumberDec(sensorVal);
  
  delay(100); // 10 Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Potentiometer**, and **TM1637 7-Segment Display** onto the canvas.
2. Connect Potentiometer **pin 1** to Arduino **5V**, **pin 2 (wiper)** to Arduino **A0**, and **pin 3** to Arduino **GND**.
3. Connect TM1637: **CLK** to Arduino **D2**, **DIO** to Arduino **D3**, **VCC** to Arduino **5V**, and **GND** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Double-click the Potentiometer, adjust its knob value, and watch the numbers update on the 7-segment display on the canvas.

## Expected Output
The TM1637 display component on the canvas lights up and displays:
```
 512
```
As you rotate the potentiometer dial, the display changes from `0` to `1023`.

## Expected Canvas Behavior

| Potentiometer Angle | Pin A0 Input | TM1637 Display Value |
| --- | --- | --- |
| 0% (Left) | 0 | `   0` |
| 50% (Center) | 512 | ` 512` |
| 100% (Right) | 1023 | `1023` |

The display updates automatically every 100 ms.

## Code Walkthrough

| Class / Function | What It Does |
| --- | --- |
| `TM1637Display display(CLK, DIO)` | Instantiates the display object specifying the Clock and Data interface pins. |
| `display.setBrightness(5)` | Adjusts the LED segment brightness level. |
| `display.showNumberDec(sensorVal)` | Converts the integer value to segment signals and prints it. |

## Hardware & Safety Concept: How 7-Segment Displays Work
A 4-digit 7-segment display contains 32 individual LEDs (4 digits x 8 segments each, including the decimal points/colons). 
- Wiring all 32 LEDs directly to the Arduino would require 32 pins, which is more than the Uno has.
- The **TM1637** chip acts as an integrated driver. It multiplexes the display, rapidly switching each digit ON and OFF one by one at high speed (invisible to the human eye). This allows full control over the 4 digits using only **2 control pins** (CLK and DIO) instead of 32.

## Try This! (Challenges)
1. **Brightness dial**: Modify the code to map the pot value (0-1023) to display brightness (0-7), so turning the dial makes the display brighter or dimmer.
2. **Voltage Display**: Convert the pot reading to display standard volts (e.g. displaying `5` for 5V, or `2` for 2.5V).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| TM1637 remains black | CLK/DIO pins swapped or missing power | Verify that CLK connects to D2 and DIO connects to D3. Ensure VCC is on 5V and GND is connected. |
| Display numbers are flickering | Too many clear statements | Avoid calling `display.clear()` inside the fast `loop()`. Only write values using `display.showNumberDec()`, which handles overwriting automatically. |

## Mode Notes
These patterns (reading analog inputs and updating the TM1637 display) are supported by MbedO interpreted mode.

## Related Projects
- [14 - Dial LED Brightness](../beginner/14-dial-led-brightness.md)
- [79 - Button Counter Display](79-button-counter-display.md)
