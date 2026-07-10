# 24 - Rain Alert

Detect rainfall using a Rain Sensor's digital output and turn on a warning LED.

## Goal
Learn how to read a rain sensor's digital threshold output and trigger an LED indicator when water is detected.

## What You Will Build
When water drops on the rain sensor board, the digital output drops to `LOW` (active low), turning on the alert LED on D13. When dry, the LED turns off.

**Why D2 and D13?** Pin D2 reads the digital threshold alarm output from the comparator chip. Pin D13 drives the alert LED.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Rain Sensor | `rain_sensor` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Rain Sensor | VCC | 5V | Power supply (5V) |
| Rain Sensor | DO (Digital Out) | D2 | Digital signal connection |
| Rain Sensor | GND | GND | Ground reference |
| LED | A (Anode, longer leg) | D13 | Output signal connection |
| LED | C (Cathode, shorter leg) | GND | Ground reference |

## Code
```cpp
const int RAIN_DO_PIN = 2;
const int LED_PIN     = 13;

void setup() {
  pinMode(RAIN_DO_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Rain Alarm System Ready");
}

void loop() {
  int rainDetected = digitalRead(RAIN_DO_PIN);
  
  // Most rain sensor comparator boards output LOW when wet
  if (rainDetected == LOW) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("[WARNING] Rain Detected! LED Active");
  } else {
    digitalWrite(LED_PIN, LOW);
  }
  
  delay(300); // Poll 3 times per second
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Rain Sensor**, and **LED** onto the canvas.
2. Connect Rain Sensor **VCC** to Arduino **5V**, **DO** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect LED **A** (Anode) to Arduino **D13** and LED **C** (Cathode) to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Double-click the Rain Sensor on the canvas to open its properties, adjust the "wetness" slider so it crosses the threshold, and watch the LED turn on.

## Expected Output

Terminal:
```
Rain Alarm System Ready
[WARNING] Rain Detected! LED Active
[WARNING] Rain Detected! LED Active
...
```

### Expected Canvas Behavior

| Moisture Condition | Sensor Grid state | Pin D2 Reading | LED State (D13) |
| --- | --- | --- | --- |
| Dry | Open | HIGH (5V) | LOW - OFF |
| Wet (Rain) | Conductive | LOW (0V) | HIGH - ON |

The LED on the canvas activates immediately when the wetness slider crosses the threshold.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `int rainDetected = digitalRead(RAIN_DO_PIN)` | Reads the digital state of pin D2, which corresponds to the threshold status output from the comparator. |
| `if (rainDetected == LOW)` | Active-low logic check. When dry, the sensor board outputs 5V (HIGH). When wet, the onboard comparator pulls the line to Ground (LOW). |

## Hardware & Safety Concept: Resistive Rain Sensors
A **resistive rain sensor** consists of a grid of nickel traces on a PCB. When water falls on the board, it acts as a conductor between positive and negative traces, lowering the overall resistance. The module's comparator chip (typically LM393) compares this resistance to a threshold set by an onboard potentiometer. If the wetness exceeds the threshold, the digital output pin `DO` drops to `LOW`.

## Try This! (Challenges)
1. **Audible Rain Alarm**: Wire a Buzzer to D8 and generate a double-chirp alarm tone (`tone(8, 1000, 100); delay(150); tone(8, 1000, 100);`) only when rain is first detected.
2. **Reverse Output**: Adjust the code to work with a sensor configuration that outputs `HIGH` when wet.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays on even when dry | Sensitivity threshold is too high | On physical modules, rotate the onboard potentiometer. On the canvas, ensure the slider wetness value is set to 0. |
| Nothing happens when wetness slider is moved | Wrong signal pin wired | Ensure you wired `DO` (Digital Out) to D2, not `AO` (Analog Out). |

## Mode Notes
These patterns (digital input polling and active-low checking) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [11 - Button ON/OFF](11-button-onoff.md)
- [25 - Rain Level Print](25-rain-level-print.md)
- [40 - Water Pump Control](../intermediate/40-water-pump-control.md)
