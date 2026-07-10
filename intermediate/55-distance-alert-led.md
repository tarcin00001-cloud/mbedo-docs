# 55 - Distance Alert LED

Trigger a warning LED immediately when an object is detected closer than 20 cm.

## Goal
Learn how to use an ultrasonic distance sensor (HC-SR04) to control a digital warning indicator (LED) using conditional comparison threshold logic.

## What You Will Build
The Arduino continuously measures distance. When an object gets closer than 20 cm, the red warning LED connected to D13 turns ON immediately. When the object moves away, the LED turns OFF.

**Why D2, D3, and D13?** Pins D2/D3 measure the distance. Pin D13 controls the warning indicator LED.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-SR04 Ultrasonic | `hc_sr04` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 Sensor | VCC | 5V | Power supply (5V) |
| HC-SR04 Sensor | TRIG | D2 | Trigger connection |
| HC-SR04 Sensor | ECHO | D3 | Echo connection |
| HC-SR04 Sensor | GND | GND | Ground reference |
| LED | A (Anode, longer leg) | D13 | Output signal connection |
| LED | C (Cathode, shorter leg) | GND | Ground reference |

## Code
```cpp
const int TRIG_PIN = 2;
const int ECHO_PIN = 3;
const int LED_PIN  = 13;

// Alert threshold distance in centimeters
const int ALERT_DISTANCE = 20;

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  
  Serial.begin(9600);
  Serial.println("Distance Alert System Ready");
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
  
  Serial.print("Distance: ");
  Serial.print(distance);
  Serial.println(" cm");
  
  // Trigger warning LED if object is closer than 20 cm
  if (distance > 0 && distance < ALERT_DISTANCE) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("[ALERT] Object too close!");
  } else {
    digitalWrite(LED_PIN, LOW);
  }
  
  delay(100); // 10 Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-SR04 Ultrasonic Sensor**, and **LED** onto the canvas.
2. Connect HC-SR04 **VCC** to Arduino **5V**, **TRIG** to Arduino **D2**, **ECHO** to Arduino **D3**, and **GND** to Arduino **GND**.
3. Connect LED **A** (Anode) to Arduino **D13** and LED **C** (Cathode) to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the HC-SR04 sensor, adjust the distance slider below 20 cm, and watch the LED turn on.

## Expected Output

Terminal:
```
Distance Alert System Ready
Distance: 55 cm
Distance: 12 cm
[ALERT] Object too close!
...
```

### Expected Canvas Behavior

| Distance Slider Input | Reading vs Limit | Pin D13 Output | LED State |
| --- | --- | --- | --- |
| Safe (> 20 cm) | Above | LOW (0V) | OFF |
| Close (< 20 cm) | Below | HIGH (5V) | ON |

The LED lights up immediately when the distance slider falls below 20 cm.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `if (distance > 0 && distance < ALERT_DISTANCE)` | Checks if the object is closer than 20 cm. The `distance > 0` condition filters out failed readings (which return 0). |

## Hardware & Safety Concept: Sensor Blind Zones
Ultrasonic sensors have a physical **blind zone** (typically 2 cm to 3 cm). If an object is closer than 2 cm, the echo returns so fast that the receiver cannot distinguish it from the outgoing burst, resulting in a failed reading (0 microseconds echo duration). In your safety code, always filter out `0` to prevent false "clear path" indications.

## Try This! (Challenges)
1. **Calibrate Range**: Change the warning threshold so that the LED lights up at `50` cm instead.
2. **Reverse logic (Limit indicator)**: Modify the code so the LED remains ON normally, and turns OFF only when an object gets too close (fail-safe indicator).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays on constantly | Sensor returns 0 continuously | A return value of 0 is treated as < 20. Ensure you have the `distance > 0` guard check in your code and check wiring. |
| LED never turns on | Wrong signal pin wired | Ensure TRIG goes to D2 and ECHO goes to D3. |

## Mode Notes
These patterns (ultrasonic measurements driving conditional digital outputs) are supported by MbedO interpreted mode.

## Related Projects
- [54 - Distance Serial](54-distance-serial.md)
- [56 - Parking Beeper](56-parking-beeper.md)
