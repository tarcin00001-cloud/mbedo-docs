# 21 - Motion Alert Print

Detect movement using a PIR Motion Sensor and print alerts to the Terminal.

## Goal
Learn how to read a PIR (Passive Infrared) motion sensor as a digital input and print a status message only when motion starts or stops.

## What You Will Build
When the PIR sensor detects motion, an alert message is printed to the Terminal. When the motion ceases, a quiet message is printed.

**Why D2?** Pin D2 supports digital input and hardware state checking, making it a standard choice for motion events.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| PIR Sensor | VCC | 5V | Power supply (5V) |
| PIR Sensor | OUT | D2 | Digital signal connection |
| PIR Sensor | GND | GND | Ground reference |

## Code
```cpp
const int PIR_PIN = 2;

int lastMotionState = LOW; // Track the previous state of the PIR sensor

void setup() {
  pinMode(PIR_PIN, INPUT); // PIR outputs digital HIGH/LOW
  Serial.begin(9600);
  Serial.println("PIR Motion Monitor Ready");
}

void loop() {
  int motionState = digitalRead(PIR_PIN);
  
  // If the motion state has changed from the previous loop run
  if (motionState != lastMotionState) {
    if (motionState == HIGH) {
      Serial.println("[ALERT] Motion Detected!");
    } else {
      Serial.println("[INFO] Motion Ended.");
    }
    // Update the state history
    lastMotionState = motionState;
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **PIR Motion Sensor** onto the canvas.
2. Connect PIR **VCC** to Arduino **5V**, **OUT** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Click the PIR sensor on the canvas to toggle its "Motion" property ON and OFF, and watch the alerts appear.

## Expected Output

Terminal:
```
PIR Motion Monitor Ready
[ALERT] Motion Detected!
[INFO] Motion Ended.
...
```

### Expected Canvas Behavior

| Canvas Motion Property | Pin D2 Reading | comparison check | Terminal Output |
| --- | --- | --- | --- |
| False (Idle) | LOW (0V) | state changed | "[INFO] Motion Ended." |
| True (Moving) | HIGH (5V) | state changed | "[ALERT] Motion Detected!" |

Output is triggered exactly once per motion state shift.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `int motionState = digitalRead(PIR_PIN)` | Reads the digital output of the PIR sensor (HIGH = motion, LOW = quiet). |
| `if (motionState != lastMotionState)` | Checks for a state transition to prevent printing continuous messages while the state remains the same. |

## Hardware & Safety Concept: Passive Infrared (PIR)
A **PIR sensor** works by measuring infrared (heat) radiation from objects in its field of view. All warm bodies emit infrared radiation. When a person or animal moves in front of the sensor, it detects the change in infrared heat levels across its dual sensor slots and drives the output pin `HIGH`.

## Try This! (Challenges)
1. **LED Alarm**: Wire an LED to D13 and modify the code so that the LED turns ON when motion is detected and OFF when it ends.
2. **Move Counter**: Add an integer variable to count the total number of movements detected, and print it alongside each alert.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Terminal floods with alerts repeatedly | Missing state change check | Make sure you compare `motionState != lastMotionState` and save the state with `lastMotionState = motionState`. |
| Sensor doesn't trigger | VCC or GND wire missing | The PIR sensor requires 5V power to operate. Verify VCC and GND connections. |

## Mode Notes
These patterns (digital reads and state change checking) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [11 - Button ON/OFF](11-button-onoff.md)
- [30 - Motion Light](../intermediate/30-motion-light.md)
