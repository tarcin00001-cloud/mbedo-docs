# 116 - ESP32 LED Bar Graph Level Indicator

Build an analog level visualizer that reads a potentiometer voltage and lights up a corresponding number of segments on a 10-LED bar graph display.

## Goal
Learn how to control a multi-pin LED array, map analog input ranges to discrete output steps, and use loops to update pin states.

## What You Will Build
A potentiometer is connected to GPIO 34. A 10-segment LED bar graph (or 10 individual LEDs) is connected to ten GPIO pins (12, 13, 14, 15, 25, 26, 27, 32, 33, 4). The number of active LEDs on the bar graph corresponds to the position of the potentiometer slider (from 0 to 10 segments lit).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| LED Bar Graph (10 segments) | `led_bar_graph` | Yes | Yes |
| 330 Ω Resistors (10) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Wiper | GPIO34 | Yellow | Analog input |
| Potentiometer | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| LED Bar Segment 1 | Anode | GPIO12 via 330 Ω | Orange | Segment controls |
| LED Bar Segment 2 | Anode | GPIO13 via 330 Ω | Orange | Segment controls |
| LED Bar Segment 3 | Anode | GPIO14 via 330 Ω | Orange | Segment controls |
| LED Bar Segment 4 | Anode | GPIO15 via 330 Ω | Orange | Segment controls |
| LED Bar Segment 5 | Anode | GPIO25 via 330 Ω | Orange | Segment controls |
| LED Bar Segment 6 | Anode | GPIO26 via 330 Ω | Orange | Segment controls |
| LED Bar Segment 7 | Anode | GPIO27 via 330 Ω | Orange | Segment controls |
| LED Bar Segment 8 | Anode | GPIO32 via 330 Ω | Orange | Segment controls |
| LED Bar Segment 9 | Anode | GPIO33 via 330 Ω | Orange | Segment controls |
| LED Bar Segment 10| Anode | GPIO4 via 330 Ω | Orange | Segment controls |
| LED Bar Cathodes | Common Ground | GND | Black | Common ground rail |

> **Wiring tip:** Each segment of the LED bar graph behaves like a standard LED and requires its own 330 Ω series resistor to protect the ESP32 GPIO from drawing too much current. Connect all cathodes to the common GND rail.

## Code
```cpp
// LED Bar Graph Level Indicator
const int POT_PIN = 34;

// Array containing the 10 GPIO pins driving the bar graph segments
const int BAR_PINS[] = {12, 13, 14, 15, 25, 26, 27, 32, 33, 4};
const int NUM_SEGS = 10;

void setup() {
  Serial.begin(115200);
  
  // Initialize all bar graph pins as outputs
  for (int i = 0; i < NUM_SEGS; i++) {
    pinMode(BAR_PINS[i], OUTPUT);
    digitalWrite(BAR_PINS[i], LOW); // Turn off all segments initially
  }
  
  Serial.println("LED Bar Graph Indicator ready.");
}

void loop() {
  int raw = analogRead(POT_PIN);
  
  // Map 12-bit ADC (0-4095) to 11 discrete levels (0 to 10 segments lit)
  int level = map(raw, 0, 4095, 0, NUM_SEGS);
  level = constrain(level, 0, NUM_SEGS);
  
  Serial.print("Pot raw: "); Serial.print(raw);
  Serial.print(" | Active Segments: "); Serial.println(level);
  
  // Update LED states
  for (int i = 0; i < NUM_SEGS; i++) {
    if (i < level) {
      digitalWrite(BAR_PINS[i], HIGH); // Light up segment
    } else {
      digitalWrite(BAR_PINS[i], LOW);  // Turn off segment
    }
  }
  
  delay(50); // 20Hz update rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer**, and **LED Bar Graph** onto the canvas.
2. Connect the 10 segments to the GPIOs as listed. Wire the Potentiometer wiper to **GPIO34**.
3. Paste the code and click **Run**.
4. Slide the potentiometer widget on the canvas. Watch the segments of the LED bar graph light up proportionally.

## Expected Output
Serial Monitor:
```
LED Bar Graph Indicator ready.
Pot raw: 0 | Active Segments: 0
Pot raw: 2048 | Active Segments: 5
Pot raw: 4095 | Active Segments: 10
```

## Expected Canvas Behavior
* Sliding the potentiometer slider completely left turns off all segments.
* Sliding the potentiometer to the middle lights up 5 segments.
* Sliding the potentiometer completely right lights up all 10 segments.
* The update response is instantaneous.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `BAR_PINS[]` | Stores the output pin map array so we can loop through them easily. |
| `map(raw, 0, 4095, 0, 10)` | Maps the analog range to 11 levels (0, 1, 2... up to 10). |
| `i < level` | Turns ON the segment if its index is lower than the active level, and OFF otherwise. |

## Hardware & Safety Concept: Current Limitations in Multi-LED Arrays
Individual ESP32 pins are rated to output a maximum of 40 mA, but the total combined current drawn from all GPIO pins simultaneously should not exceed **120 mA** to prevent internal thermal damage to the microchip silicon. Driving 10 LEDs at 10 mA each draws 100 mA, which is close to this limit. Using 330 Ω series resistors restricts current to approx 5 mA per pin, keeping the total consumption to a safe 50 mA.

## Try This! (Challenges)
1. **Dynamic Dot Mode**: Light up only the single active segment representing the level, keeping all other segments OFF (simulating a dot-indicator rather than a bar).
2. **Reverse direction**: Reverse the mapping so turning the potentiometer to the left increases the bar graph length.
3. **Sound level visualizer**: Connect the Sound Sensor (Project 96) and map its peak-to-peak volume to the bar graph instead of the potentiometer.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Individual segments stay off | Resistor disconnected or pin mapping mismatched | Check individual wire connections; verify index pin assignments in the `BAR_PINS` array |
| LEDs are dim | Series resistor value too high | Ensure 330 Ω or 220 Ω resistors are utilized, not 10 kΩ |
| Graph flickers | Potentiometer input noise | Smooth the ADC reading by taking an average of 5 samples |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [32 - ESP32 Potentiometer LED Dimmer](../beginner/32-esp32-potentiometer-dimmer-control.md) (Note: Potentiometer LED Dimmer)
- [96 - ESP32 Sound Level OLED Decibel Meter](96-esp32-sound-level-oled-decibel-meter.md)
- [04 - ESP32 Multiple LED Chase](../beginner/04-esp32-multiple-led-chase.md)
