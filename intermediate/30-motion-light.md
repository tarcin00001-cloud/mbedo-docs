# 30 - Motion Light

Turn a light on automatically when motion is detected, using a PIR sensor.

## Goal
Learn how to use a digital input sensor (PIR) to control a digital output actuator (LED) using basic conditional threshold logic.

## What You Will Build
When the PIR sensor detects motion, the LED connected to D13 turns ON immediately. When the motion stops, the LED turns OFF.

**Why D2 and D13?** Pin D2 reads the digital logic high/low from the PIR sensor. Pin D13 drives the onboard indicator LED representing our automatic lamp.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| PIR Sensor | VCC | 5V | Power supply (5V) |
| PIR Sensor | OUT | D2 | Digital signal connection |
| PIR Sensor | GND | GND | Ground reference |
| LED | A (Anode, longer leg) | D13 | Output signal connection |
| LED | C (Cathode, shorter leg) | GND | Ground reference |

## Code
```cpp
const int PIR_PIN = 2;
const int LED_PIN = 13;

void setup() {
  pinMode(PIR_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  
  Serial.begin(9600);
  Serial.println("Motion Light Controller Ready");
}

void loop() {
  int motionDetected = digitalRead(PIR_PIN);
  
  // HIGH means motion is currently detected
  if (motionDetected == HIGH) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("Motion Detected - Light ON");
  } else {
    digitalWrite(LED_PIN, LOW);
  }
  
  delay(100); // 10 Hz poll rate
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **PIR Motion Sensor**, and **LED** onto the canvas.
2. Connect PIR **VCC** to Arduino **5V**, **OUT** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect LED **A** (Anode) to Arduino **D13** and LED **C** (Cathode) to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Click the PIR sensor on the canvas to toggle its "Motion" property ON and OFF, and watch the LED respond.

## Expected Output

Terminal:
```
Motion Light Controller Ready
Motion Detected - Light ON
Motion Detected - Light ON
...
```

### Expected Canvas Behavior

| PIR Sensor State | Pin D2 Reading | Pin D13 Output | LED State |
| --- | --- | --- | --- |
| Quiet (No Motion) | LOW (0V) | LOW (0V) | OFF |
| Active (Motion) | HIGH (5V) | HIGH (5V) | ON |

The LED remains lit as long as motion is detected by the PIR sensor.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `int motionDetected = digitalRead(PIR_PIN)` | Reads the digital state of the PIR sensor (HIGH when active). |
| `if (motionDetected == HIGH)` | Triggers the LED pin HIGH if motion is present; otherwise, it pulls it LOW. |

## Hardware & Safety Concept: Sensor Stabilization
Real PIR sensors require a **stabilization time** (typically 30 to 60 seconds) after powering up to calibrate against the ambient infrared background of the room. During this calibration period, the sensor may trigger false HIGH outputs. In your code, you can bypass this by adding a startup delay in `setup()` if needed.

## Try This! (Challenges)
1. **Off-Delay Timer**: Modify the code using `delay()` so that once motion is detected, the LED stays on for exactly 3 seconds (3000 ms) before turning off, even if the motion sensor goes quiet immediately.
2. **Reverse Light**: Modify the code so that the LED is normally ON, and turns OFF only when motion is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays on constantly | Floating inputs | Ensure the PIR output pin is firmly wired to D2 and the GND pin is connected to ground. |
| LED never turns on | Sensor has no power | Verify the PIR VCC connects to 5V and GND connects to GND. |

## Mode Notes
These patterns (digital input polling driving digital outputs) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [21 - Motion Alert Print](../beginner/21-motion-alert-print.md)
- [31 - Motion Alarm](31-motion-alarm.md)
- [39 - Auto Lamp Relay](39-auto-lamp-relay.md)
