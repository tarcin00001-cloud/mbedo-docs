# 39 - ESP32 Rain Sensor Digital Alert

Detect the presence of rain or moisture using a rain sensor module and trigger a digital alert when water contacts the sensing pad.

## Goal
Learn how a rain sensor module converts water conductivity into a digital signal, how to read its DO (digital output) pin, and how to drive a warning output when rain is detected.

## What You Will Build
A rain sensor module connected to GPIO 4. When rain (or water) bridges the sensor pad traces, the module drives DO LOW. The ESP32 detects this and lights a blue LED on GPIO 5 while printing a rain alert.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Rain Sensor Module (FC-37 or YL-83) | `rain_sensor` | Yes | Yes |
| LED (blue) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Rain Sensor Module | VCC | 3V3 | Red | Module supply voltage |
| Rain Sensor Module | GND | GND | Black | Module ground |
| Rain Sensor Module | DO (digital out) | GPIO4 | Yellow | Active-LOW output — LOW when wet |
| LED (blue) | Anode (+) | GPIO5 via 330 Ω | Blue | Rain alert indicator |
| LED (blue) | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** The rain sensor module has a sensitivity trimmer potentiometer. Turn it clockwise to increase sensitivity (trigger on light drizzle) or counter-clockwise to require heavier rainfall. The DO pin uses active-LOW logic: LOW = rain detected, HIGH = dry. Keep the control board (with the potentiometer) separate from the sensing pad — only the sensing pad should be exposed to rain.

## Code
```cpp
// Rain Sensor Digital Alert — active-LOW detection
const int RAIN_PIN = 4;
const int LED_PIN  = 5;

void setup() {
  pinMode(RAIN_PIN, INPUT);    // Module drives pin actively
  pinMode(LED_PIN,  OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Serial.begin(115200);
  Serial.println("Rain Sensor ready — monitoring weather.");
}

void loop() {
  int rainState = digitalRead(RAIN_PIN);

  // Active-LOW: LOW = rain detected, HIGH = dry
  if (rainState == LOW) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("!! RAIN DETECTED — Alert !!");
  } else {
    digitalWrite(LED_PIN, LOW);
    Serial.println("Dry — no rain.");
  }

  delay(500);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Rain Sensor**, and **LED** onto the canvas.
2. Connect Rain Sensor **DO** to **GPIO4**, LED **anode** to **GPIO5**.
3. Paste the code and click **Run**.
4. Toggle the rain sensor widget to simulate wet conditions.

## Expected Output
Serial Monitor:
```
Rain Sensor ready — monitoring weather.
Dry — no rain.
Dry — no rain.
!! RAIN DETECTED — Alert !!
!! RAIN DETECTED — Alert !!
Dry — no rain.
```

## Expected Canvas Behavior
* LED off when the rain sensor widget is in the dry state (DO = HIGH).
* LED lights when the rain sensor widget is toggled to wet (DO = LOW).
* Serial Monitor updates every 500 ms.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `int rainState = digitalRead(RAIN_PIN)` | Reads the module DO pin: LOW = rain, HIGH = dry. |
| `if (rainState == LOW)` | Active-LOW detection: water bridging the pad lowers DO to GND. |
| `digitalWrite(LED_PIN, HIGH)` | Lights the blue alert LED when rain is present. |
| `delay(500)` | 2 Hz polling — rain is a slow-changing event; 500 ms is adequate. |

## Hardware & Safety Concept: Resistive Rain Sensing
A rain sensor pad consists of a set of interleaved bare copper traces on a PCB. In dry conditions there is no conductive path between the traces (infinite resistance). When water drops land on the pad, water's dissolved minerals give it slight conductivity. This bridges the interleaved traces, reducing the resistance across the pad to a few kΩ. The module's comparator circuit senses this resistance drop and triggers the DO output LOW. Rain sensors are used in: **automatic car windshield wipers**, **skylight/window auto-closers**, **smart irrigation systems** (stop watering when it rains), and **weather station nodes**. Their limitation is that very pure distilled water has very low conductivity and may not trigger the sensor; real rainwater always contains enough dissolved ions to conduct reliably.

## Try This! (Challenges)
1. **Rain + LDR combination**: Trigger an alert only when it is both raining AND dark (combine Projects 34 and 39).
2. **Buzzer alarm**: Add a buzzer on GPIO 15 that beeps every 2 seconds while rain is detected.
3. **Analog level**: Connect the module's AO pin to GPIO 35 and print the raw conductivity value to measure rainfall intensity.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Always shows RAIN DETECTED even when dry | Sensor pad contaminated | Clean the pad with isopropyl alcohol and dry thoroughly |
| Never triggers even with water | Sensitivity too low | Rotate the trimmer potentiometer clockwise to increase sensitivity |
| Module does not power on | VCC wired to 5 V GPIO (not Vin) | Move VCC wire to the 3V3 rail |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [36 - ESP32 PIR Motion Sensor Alert](36-esp32-pir-motion-sensor-alert.md)
- [40 - ESP32 Soil Moisture Analog Level Print](40-esp32-soil-moisture-analog-level-print.md)
- [41 - ESP32 Water Level Analog Level Print](41-esp32-water-level-analog-level-print.md)
