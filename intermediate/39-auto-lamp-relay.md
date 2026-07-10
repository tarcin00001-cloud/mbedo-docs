# 39 - Auto Lamp Relay

Actuate a high-power security lighting relay automatically when motion is detected.

## Goal
Learn how to use a passive motion sensor (PIR) to control a high-power electrical load through an isolation relay module.

## What You Will Build
When the PIR sensor detects motion, the relay module connected to pin D9 turns ON (triggering a floodlight or security camera in a real installation). When motion stops, the relay turns OFF.

**Why D2 and D9?** Pin D2 reads the digital motion state. Pin D9 drives the relay coil control line.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| Relay Module | `relay_module` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| PIR Sensor | VCC | 5V | Power supply (5V) |
| PIR Sensor | OUT | D2 | Digital signal connection |
| PIR Sensor | GND | GND | Ground reference |
| Relay Module | VCC | 5V | Power supply (5V) |
| Relay Module | IN | D9 | Digital control line |
| Relay Module | GND | GND | Ground reference |

## Code
```cpp
const int PIR_PIN   = 2;
const int RELAY_PIN = 9;

void setup() {
  pinMode(PIR_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with relay off
  
  Serial.begin(9600);
  Serial.println("Auto Lamp Controller Ready");
}

void loop() {
  int motionDetected = digitalRead(PIR_PIN);
  
  if (motionDetected == HIGH) {
    digitalWrite(RELAY_PIN, HIGH);
    Serial.println("Motion detected - Security Lamp Relay ON");
  } else {
    digitalWrite(RELAY_PIN, LOW);
  }
  
  delay(200); // 5 Hz poll rate
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **PIR Motion Sensor**, and **Relay Module** onto the canvas.
2. Connect PIR **VCC** to Arduino **5V**, **OUT** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect Relay **VCC** to Arduino **5V**, **IN** to Arduino **D9**, and **GND** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Click the PIR sensor on the canvas to toggle its "Motion" property ON and OFF, and watch the Relay switch.

## Expected Output

Terminal:
```
Auto Lamp Controller Ready
Motion detected - Security Lamp Relay ON
Motion detected - Security Lamp Relay ON
...
```

### Expected Canvas Behavior

| PIR Sensor State | Pin D2 Reading | Pin D9 Output | Relay Switch State |
| --- | --- | --- | --- |
| Quiet | LOW (0V) | LOW (0V) | OFF (Normally Open) |
| Active | HIGH (5V) | HIGH (5V) | ON (Closed) |

The relay module indicator switches state on the canvas when motion is detected.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `int motionDetected = digitalRead(PIR_PIN)` | Reads the digital motion state (HIGH = moving). |
| `digitalWrite(RELAY_PIN, HIGH)` | Pushes 5V to D9, closing the relay contacts to turn on high-power lighting. |

## Hardware & Safety Concept: Relay Load Terminals
Relay modules have three screw terminals on the high-power side:
- **COM (Common)**: Connects to the incoming hot line.
- **NO (Normally Open)**: Disconnected by default. Closes the circuit when the relay is energized. Best for security lights.
- **NC (Normally Closed)**: Connected by default. Opens the circuit when energized.

## Try This! (Challenges)
1. **Light Extender**: Add a delay so that once motion is detected, the relay stays closed for exactly 5 seconds (5000 ms) before turning off.
2. **Reverse Safety Light**: Configure the relay to turn OFF when motion is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay switches constantly | Optical noise or floating ground | Ensure all grounds are connected together. |
| Relay clicks but no power to the lamp | Wiring error on output terminals | Ensure the load is wired through COM and NO, not NC. |

## Mode Notes
These patterns (digital input polling driving digital relay outputs) are supported by MbedO interpreted mode.

## Related Projects
- [30 - Motion Light](30-motion-light.md)
- [35 - Flame Relay](35-flame-relay.md)
- [37 - Night Fan Relay](37-night-fan-relay.md)
