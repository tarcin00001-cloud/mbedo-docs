# 45 - ESP32 Hall Effect Magnetic Detector

Use a Hall effect sensor (A3144 or SS49E module) to detect the presence of a magnetic field and print the detection state to the Serial Monitor.

## Goal
Learn how a Hall effect sensor converts magnetic field strength into a digital signal, and how to use it as a contactless magnetic proximity detector.

## What You Will Build
A Hall effect sensor module on GPIO 4. When a magnet is brought close, the sensor's open-drain output is pulled LOW. The ESP32 detects this with `INPUT_PULLUP` and lights an LED on GPIO 5 while printing a detection message.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Hall Effect Sensor Module (A3144 / KY-024) | `hall_sensor` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Hall Sensor Module | VCC | 3V3 | Red | Module supply voltage |
| Hall Sensor Module | GND | GND | Black | Module ground |
| Hall Sensor Module | DO (digital out) | GPIO4 | Yellow | Active-LOW output — LOW when magnet present |
| LED | Anode (+) | GPIO5 via 330 Ω | Orange | Magnetic detection indicator |
| LED | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** The A3144 Hall effect sensor uses an open-drain output — it pulls its output LOW when the south pole of a magnet faces the sensor face, and releases it (floating HIGH with pull-up) when the magnet is removed. Use `INPUT_PULLUP` on GPIO 4 to provide the pull-up for the open-drain output. The KY-024 module includes an onboard pull-up and triggers on either pole.

## Code
```cpp
// Hall Effect Magnetic Field Detector — active-LOW open-drain output
const int HALL_PIN = 4;
const int LED_PIN  = 5;

void setup() {
  pinMode(HALL_PIN, INPUT_PULLUP);  // Pull-up for open-drain output
  pinMode(LED_PIN,  OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Serial.begin(115200);
  Serial.println("Hall Effect Detector ready — bring a magnet close.");
}

void loop() {
  int hallState = digitalRead(HALL_PIN);

  // Active-LOW: LOW = magnet detected, HIGH = no field
  if (hallState == LOW) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("MAGNET DETECTED — field present.");
  } else {
    digitalWrite(LED_PIN, LOW);
    Serial.println("No magnetic field.");
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Hall Sensor**, and **LED** onto the canvas.
2. Connect Hall Sensor **DO** to **GPIO4**, LED **anode** to **GPIO5**.
3. Paste the code and click **Run**.
4. Toggle the hall sensor widget to simulate bringing a magnet close.

## Expected Output
Serial Monitor:
```
Hall Effect Detector ready — bring a magnet close.
No magnetic field.
No magnetic field.
MAGNET DETECTED — field present.
MAGNET DETECTED — field present.
No magnetic field.
```

## Expected Canvas Behavior
* LED off when the hall sensor widget is inactive (no magnet).
* LED lights immediately when the hall sensor widget is activated (magnet present).
* Returns to off state when the widget is deactivated.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `pinMode(HALL_PIN, INPUT_PULLUP)` | Enables the internal pull-up — required for the open-drain output of the A3144. |
| `if (hallState == LOW)` | Active-LOW detection: sensor pulls DO to GND when a magnetic field is present. |
| `digitalWrite(LED_PIN, HIGH)` | Lights the indicator on magnetic field detection. |
| `delay(200)` | 5 Hz polling — hall effect is instantaneous; no bounce to debounce. |

## Hardware & Safety Concept: Hall Effect Sensing
The Hall effect was discovered by Edwin Hall in 1879. When a current-carrying conductor is placed in a perpendicular magnetic field, a voltage (Hall voltage) develops across the conductor at right angles to both the current and the field. In a Hall effect sensor, this Hall voltage is amplified and compared to a threshold to produce a digital output. Hall sensors are completely contactless — the magnet never touches the sensor. This makes them ideal for: **rotational speed sensing** (magnets on a wheel, sensor counts pulses), **brushless motor commutation** (detecting rotor pole position), **current sensing** (magnetic field around a conductor), **proximity switches** in sealed environments (where physical contact switches would corrode or fail). The ESP32 also has a built-in Hall sensor connected to GPIO 36/39, but its sensitivity is lower than a dedicated module.

## Try This! (Challenges)
1. **RPM counter**: Attach a magnet to a rotating wheel; count hall sensor pulses per second with `millis()` to calculate RPM.
2. **Polarity detector**: Use the KY-024's AO pin and ADC to detect both north and south poles by voltage direction.
3. **Proximity switch**: Replace a mechanical door switch with a hall sensor and magnet — test the detection range by moving the magnet from 0 to 5 cm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor never triggers | Wrong magnet pole facing sensor face | Flip the magnet; A3144 triggers only on south pole |
| LED always on | Short on DO pin to GND | Check wiring; verify sensor is not defective |
| Detection range very short | Weak magnet | Use a neodymium (rare-earth) magnet instead of a ceramic ferrite magnet |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [25 - ESP32 Magnetic Switch Gate Alert](25-esp32-magnetic-switch-gate-alert.md)
- [28 - ESP32 Tilt Sensor Level Indicator](28-esp32-tilt-sensor-level-indicator.md)
- [37 - ESP32 IR Obstacle Sensor Print](37-esp32-ir-obstacle-sensor-print.md)
