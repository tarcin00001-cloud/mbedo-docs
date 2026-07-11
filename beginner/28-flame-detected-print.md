# 28 - Flame Detected Print

Detect fire hazards using an Infrared Flame Sensor and display alerts in the Terminal.

## Goal
Learn how to read a digital flame sensor's threshold output and print safety alert warnings to the Terminal.

## What You Will Build
When a flame is present near the sensor, the digital output pin drops to `LOW` (active low), and the Terminal prints "[ALERT] FIRE DETECTED!". When the flame is removed, it prints "[INFO] Environment Safe.".

**Why D2?** Pin D2 reads the digital threshold alert from the comparator board.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Flame Sensor | `flame_sensor` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Flame Sensor | VCC | 5V | Power supply (5V) |
| Flame Sensor | DO (Digital Out) | D2 | Digital signal connection |
| Flame Sensor | GND | GND | Ground reference |

## Code
```cpp
const int FLAME_PIN = 2;

int lastFlameState = HIGH; // Sensor outputs HIGH when safe

void setup() {
  pinMode(FLAME_PIN, INPUT);
  Serial.begin(9600);
  Serial.println("Flame Monitor Ready");
}

void loop() {
  int flameState = digitalRead(FLAME_PIN);

  // Report the current sensor state. The module uses active-low output.
  if (flameState == LOW) {
    Serial.println("Flame detected - sensor output is LOW");
  } else {
    Serial.println("No flame detected - environment is safe");
  }
  
  // Trigger actions only on state change
  if (flameState != lastFlameState) {
    if (flameState == LOW) {
      Serial.println("[ALERT] FIRE DETECTED!");
    } else {
      Serial.println("[INFO] Environment Safe.");
    }
    lastFlameState = flameState;
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **Flame Sensor** onto the canvas.
2. Connect Flame Sensor **VCC** to Arduino **5V**, **DO** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Double-click the Flame Sensor, adjust the "flame" property slider (representing fire strength) to cross the threshold, and watch the alerts appear.

## Expected Output

Terminal:
```
Flame Monitor Ready
[ALERT] FIRE DETECTED!
[INFO] Environment Safe.
...
```

### Expected Canvas Behavior

| Flame Status | DO Pin State (D2) | Comparison Check | Terminal Printout |
| --- | --- | --- | --- |
| Absent (Safe) | HIGH (5V) | state changed | "[INFO] Environment Safe." |
| Present (Fire) | LOW (0V) | state changed | "[ALERT] FIRE DETECTED!" |

Alerts print exactly once when the flame condition starts or ends.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `int flameState = digitalRead(FLAME_PIN)` | Reads the digital state of pin D2. |
| `if (flameState == LOW)` | Active-low logic check. When the sensor detects the specific infrared wavelength of a flame, the onboard comparator pulls the digital output LOW. |

## Hardware & Safety Concept: Infrared Photodiodes
An **infrared flame sensor** uses a high-speed photodiode sensitive to infrared wavelengths in the range of 760 nm to 1100 nm (the spectrum emitted by open flames). Since daylight and standard lightbulbs also emit infrared waves, physical sensors include a potentiometer to adjust sensitivity and prevent false triggers.

## Try This! (Challenges)
1. **Siren Alert**: Add a Buzzer to D8 and code a fast-pulsing siren tone (`tone(8, 1500); delay(100); tone(8, 500); delay(100);`) that triggers only while `flameState == LOW`.
2. **Sprinkler Switch**: Add a Relay Module to D9 and code it to turn ON (`HIGH`) only when a flame is detected, representing a sprinkler system.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Terminal prints warnings continuously | High noise or floating connection | Check the digital wire on D2. Ensure VCC and GND are connected firmly. |
| Flame is not detected | Slider value too low | Adjust the flame slider properties on the canvas to exceed the detection threshold. |

## Mode Notes
These patterns (digital input polling and conditional serial logging) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [21 - Motion Alert Print](21-motion-alert-print.md)
- [34 - Flame Alarm](../intermediate/34-flame-alarm.md)
