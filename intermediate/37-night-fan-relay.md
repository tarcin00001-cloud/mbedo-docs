# 37 - Night Fan Relay

Actuate a ventilation fan relay automatically when it becomes dark.

## Goal
Learn how to use an analog light sensor (LDR) to switch a high-power relay module based on light thresholds.

## What You Will Build
When the ambient light level falls below 400 (simulating night), the relay module connected to pin D9 turns ON (triggering a fan or heater in a real installation). When it gets bright, the relay turns OFF.

**Why A0 and D9?** Pin A0 reads the analog ambient light level. Pin D9 drives the relay control signal.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| LDR Light Sensor | `ldr` | Yes | Yes |
| Relay Module | `relay_module` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| LDR Sensor | VCC | 5V | Power supply (5V) |
| LDR Sensor | AO | A0 | Analog signal connection |
| LDR Sensor | GND | GND | Ground reference |
| Relay Module | VCC | 5V | Power supply (5V) |
| Relay Module | IN | D9 | Digital control line |
| Relay Module | GND | GND | Ground reference |

## Code
```cpp
const int LDR_PIN   = A0;
const int RELAY_PIN = 9;

// Threshold for night activation
const int DARK_THRESHOLD = 400;

void setup() {
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with relay off
  
  Serial.begin(9600);
  Serial.println("Night Fan Controller Ready");
}

void loop() {
  int lightLevel = analogRead(LDR_PIN);
  
  Serial.print("Light Level: ");
  Serial.println(lightLevel);
  
  if (lightLevel < DARK_THRESHOLD) {
    digitalWrite(RELAY_PIN, HIGH);
    Serial.println("Night detected - Fan Relay ON");
  } else {
    digitalWrite(RELAY_PIN, LOW);
  }
  
  delay(500); // Check twice per second
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **LDR Light Sensor**, and **Relay Module** onto the canvas.
2. Connect LDR **VCC** to Arduino **5V**, **AO** to Arduino **A0**, and **GND** to Arduino **GND**.
3. Connect Relay **VCC** to Arduino **5V**, **IN** to Arduino **D9**, and **GND** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the LDR Sensor, adjust the slider to a low light level (below 400), and watch the Relay switch on.

## Expected Output

Terminal:
```
Night Fan Controller Ready
Light Level: 850
Light Level: 320
Night detected - Fan Relay ON
...
```

### Expected Canvas Behavior

| Ambient Light | Pin A0 Reading | Pin D9 Output | Relay Switch State |
| --- | --- | --- | --- |
| Bright | > 400 | LOW (0V) | OFF (Normally Open) |
| Dark | < 400 | HIGH (5V) | ON (Closed) |

The relay module changes state on the canvas when the light level falls below the threshold.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `int lightLevel = analogRead(LDR_PIN)` | Measures the voltage from the LDR module. |
| `digitalWrite(RELAY_PIN, HIGH)` | Drives D9 HIGH to close the relay contacts. |

## Hardware & Safety Concept: Inductive Loads & Flyback Diodes
Devices with electric motors (like fans or pumps) are **inductive loads**. When you cut power to a motor, the collapsing magnetic field inside the motor windings generates a high-voltage spike (flyback voltage) that can travel back down the wires and damage your transistors or microcontroller. Professional relay boards include a **flyback diode** (or snubber circuit) to safely redirect and dissipate this voltage spike.

## Try This! (Challenges)
1. **Inverse Control (Sunlight Fan)**: Modify the code so that the relay turns ON when it is *bright* (simulating a solar-powered fan that runs when the sun is shining).
2. **Periodic Check**: Adjust the refresh delay to 5 seconds (5000 ms) to reduce power consumption.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay clicks on and off rapidly at dusk | No hysteresis | Implement a buffer range in code or ensure the LDR is not facing the light source driven by the relay. |
| Relay doesn't respond | VCC/GND reversed | Verify all wires match the wiring diagram. |

## Mode Notes
These patterns (LDR analog check driving a digital relay pin) are supported by MbedO interpreted mode.

## Related Projects
- [19 - Night Detector](../beginner/19-night-detector.md)
- [35 - Flame Relay](35-flame-relay.md)
- [99 - Auto Night Light](../advanced/99-auto-night-light.md)
