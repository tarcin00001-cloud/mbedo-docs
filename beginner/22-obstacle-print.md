# 22 - Obstacle Print

Detect nearby obstacles using an Infrared (IR) Proximity Sensor and print status updates.

## Goal
Learn how to read a digital IR proximity sensor and log obstacle detections to the Terminal.

## What You Will Build
When an object is placed in front of the IR sensor, it triggers a digital detection signal, and the Terminal prints "[WARNING] Obstacle Detected!". When the object is removed, it prints "[INFO] Path Clear.".

**Why D2?** Pin D2 reads the digital output of the IR sensor module. D2 is used for standard digital inputs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| IR Sensor | `ir_sensor` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| IR Sensor | VCC | 5V | Power supply (5V) |
| IR Sensor | OUT | D2 | Digital signal connection |
| IR Sensor | GND | GND | Ground reference |

## Code
```cpp
const int IR_PIN = 2;

int lastObstacleState = HIGH; // Most IR sensors output HIGH by default (active LOW)

void setup() {
  pinMode(IR_PIN, INPUT);
  Serial.begin(9600);
  Serial.println("IR Obstacle Monitor Ready");
}

void loop() {
  int obstacleState = digitalRead(IR_PIN);
  
  // Detect a state transition
  if (obstacleState != lastObstacleState) {
    if (obstacleState == LOW) {
      Serial.println("[WARNING] Obstacle Detected!");
    } else {
      Serial.println("[INFO] Path Clear.");
    }
    lastObstacleState = obstacleState;
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **IR Sensor** onto the canvas.
2. Connect IR Sensor **VCC** to Arduino **5V**, **OUT** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Click the IR Sensor on the canvas to toggle the "Obstacle" property ON and OFF, and watch the output in the Terminal.

## Expected Output

Terminal:
```
IR Obstacle Monitor Ready
[WARNING] Obstacle Detected!
[INFO] Path Clear.
...
```

### Expected Canvas Behavior

| Canvas Obstacle State | Pin D2 Reading | comparison check | Terminal Output |
| --- | --- | --- | --- |
| False (Clear) | HIGH (5V) | state changed | "[INFO] Path Clear." |
| True (Blocked) | LOW (0V) | state changed | "[WARNING] Obstacle Detected!" |

Output updates only when the path status changes.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `int obstacleState = digitalRead(IR_PIN)` | Reads the digital output of the IR sensor. Standard proximity sensors output `LOW` when an obstacle is detected (active-low) and `HIGH` when clear. |

## Hardware & Safety Concept: Active Infrared Sensors
Unlike passive infrared (PIR) sensors that detect ambient heat, an **active IR sensor** emits its own infrared light beam using an IR transmitter LED. If an object is nearby, the IR light bounces off the surface and is detected by an IR receiver photodiode. The onboard comparator circuit then pulls the output pin `LOW`.

## Try This! (Challenges)
1. **Obstacle LED Alert**: Wire an LED to D13 and make it turn ON when an obstacle is detected, and OFF when the path is clear.
2. **Double Alert tone**: Play a warning tone when an obstacle is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Status bounces continuously | Pin is floating or loose wiring | Verify that the OUT pin is securely connected to D2 and the GND pin to ground. |
| Nothing is printed | Code isn't running or wrong baud rate | Check that the runtime is set to Interpreted AVR subset and the Terminal window is open at 9600 baud. |

## Mode Notes
These patterns (digital input polling and state transition checks) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [16 - Touch ON/OFF](16-touch-onoff.md)
- [44 - Obstacle Beeper](../intermediate/44-obstacle-beeper.md)
