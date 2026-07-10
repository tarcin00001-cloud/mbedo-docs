# 35 - Flame Relay

Actuate a high-power safety relay immediately when fire is detected by a flame sensor.

## Goal
Learn how to read a digital hazard sensor (Flame Sensor) and use it to control a high-voltage isolation actuator (Relay Module) using digital conditional logic.

## What You Will Build
When a flame is detected, the flame sensor outputs a digital `LOW` signal. The Arduino detects this and drives pin D9 `HIGH`, activating the Relay Module (which would turn on a water pump, fire sprinkler valve, or warning beacon in a real-world system).

**Why D2 and D9?** Pin D2 reads the digital hazard alarm. Pin D9 drives the relay control coil line.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Flame Sensor | `flame_sensor` | Yes | Yes |
| Relay Module | `relay_module` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Flame Sensor | VCC | 5V | Power supply (5V) |
| Flame Sensor | DO (Digital Out) | D2 | Digital signal connection |
| Flame Sensor | GND | GND | Ground reference |
| Relay Module | VCC | 5V | Power supply (5V) |
| Relay Module | IN (Input Signal) | D9 | Digital control line |
| Relay Module | GND | GND | Ground reference |

## Code
```cpp
const int FLAME_PIN = 2;
const int RELAY_PIN = 9;

void setup() {
  pinMode(FLAME_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Keep relay off at startup
  
  Serial.begin(9600);
  Serial.println("Flame Relay Controller Ready");
}

void loop() {
  int flameState = digitalRead(FLAME_PIN);
  
  // Active LOW: LOW means flame is detected
  if (flameState == LOW) {
    digitalWrite(RELAY_PIN, HIGH);
    Serial.println("[ALERT] FIRE! Activating Safety Relay!");
  } else {
    digitalWrite(RELAY_PIN, LOW);
  }
  
  delay(200); // 5 Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Flame Sensor**, and **Relay Module** onto the canvas.
2. Connect Flame Sensor **VCC** to Arduino **5V**, **DO** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect Relay **VCC** to Arduino **5V**, **IN** to Arduino **D9**, and **GND** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the Flame Sensor, adjust the "flame" slider property to trigger the sensor, and watch the Relay switch states on the canvas.

## Expected Output

Terminal:
```
Flame Relay Controller Ready
[ALERT] FIRE! Activating Safety Relay!
[ALERT] FIRE! Activating Safety Relay!
...
```

### Expected Canvas Behavior

| Flame Status | Pin D2 Reading | Pin D9 Output | Relay Switch State |
| --- | --- | --- | --- |
| Safe | HIGH (5V) | LOW (0V) | OFF (Normally Open) |
| Hazard (Fire) | LOW (0V) | HIGH (5V) | ON (Closed) |

The relay indicator on the canvas switches to the active closed state when a flame is detected.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `int flameState = digitalRead(FLAME_PIN)` | Reads the digital output of the flame sensor (LOW when active). |
| `digitalWrite(RELAY_PIN, HIGH)` | Drives D9 HIGH, feeding current to the relay module optocoupler/coil and closing the high-power circuit switch contacts. |

## Hardware & Safety Concept: Relay Isolation
Microcontrollers operate at low voltages (5V or 3.3V) and low currents (max 40 mA per pin). High-power devices, such as water pumps, sirens, or mains appliances (120V/240V AC), require much higher current and voltage. A **relay** is an electromagnetically operated switch. 
- Pushing 5V into the `IN` pin energizes an internal coil, which mechanically pulls a high-power switch closed.
- The low-voltage Arduino circuit is **galvanically isolated** from the high-voltage load circuit, preventing electrical feedback from destroying the microcontroller.

## Try This! (Challenges)
1. **Pulse Relay Alert**: Modify the code to toggle the relay ON and OFF every 1 second (1000 ms) while a flame is active (e.g. to pulse a hazard beacon light).
2. **Reverse Safety Switch**: Code a fail-safe system where the relay is normally closed (ON) to run a critical exhaust fan, and turns OFF only when a flame is detected to cut off fresh oxygen.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay switches on and off constantly | Pin noise or floating ground | Make sure the flame sensor and the relay share a common Ground (GND) with the Arduino. |
| Relay doesn't switch on | Inadequate power | Opto-isolated relay modules require a stable 5V line. Make sure VCC is connected to 5V, not 3.3V. |

## Mode Notes
These patterns (digital input polling driving digital outputs) are supported by MbedO interpreted mode.

## Related Projects
- [34 - Flame Alarm](34-flame-alarm.md)
- [39 - Auto Lamp Relay](39-auto-lamp-relay.md)
- [116 - Multi-Zone Fire System](../expert/116-multi-zone-fire-system.md)
