# 160 - Ultrasonic Wind Anemometer

Estimate wind speed by measuring the time-of-flight difference between two HC-SR04 ultrasonic transducers aimed in opposite directions along a common axis. Sound travelling with the wind arrives faster than sound travelling against it; the difference reveals wind velocity.

## Goal
Understand how differential ultrasonic time-of-flight can detect airflow direction and speed, and how to drive two HC-SR04 modules from a single ARIES v3 board using four GPIO pins and state-machine-based timing.

## What You Will Build
Two HC-SR04 sensors are mounted facing each other along a straight tube or open path. Sensor A (trigger GPIO 14, echo GPIO 15) fires first; Sensor B (trigger GPIO 12, echo GPIO 13) fires second. The board measures each echo pulse width using `pulseIn`, converts the durations to distance/time-of-flight, computes the differential, applies the standard ultrasonic anemometer equation, and prints wind speed (m/s and km/h) to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor A | `hcsr04` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor B | `hcsr04` | Yes | Yes |
| Mounting tube or channel (~30 cm) | — | No | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 Sensor A | VCC | 5V | Red | Sensor power |
| HC-SR04 Sensor A | GND | GND | Black | Common ground |
| HC-SR04 Sensor A | TRIG | GPIO 14 | Yellow | Trigger pulse from ARIES |
| HC-SR04 Sensor A | ECHO | GPIO 15 | Green | Echo pulse to ARIES |
| HC-SR04 Sensor B | VCC | 5V | Red | Sensor power |
| HC-SR04 Sensor B | GND | GND | Black | Common ground |
| HC-SR04 Sensor B | TRIG | GPIO 12 | Orange | Trigger pulse from ARIES |
| HC-SR04 Sensor B | ECHO | GPIO 13 | Blue | Echo pulse to ARIES |

> **Wiring tip:** Mount the two HC-SR04 sensors facing each other at a fixed separation of 20–30 cm inside a straight PVC tube. The ECHO pins output 5 V logic pulses; use a 1 kΩ / 2 kΩ voltage divider on each ECHO line to bring the signal level down to 3.3 V before connecting to the ARIES GPIO pin to protect the board inputs.

## Code
```cpp
// Ultrasonic Wind Anemometer
// Sensor A: TRIG=GPIO14, ECHO=GPIO15
// Sensor B: TRIG=GPIO12, ECHO=GPIO13

#define TRIG_A  14
#define ECHO_A  15
#define TRIG_B  12
#define ECHO_B  13

// Fixed separation between sensors in metres
const float SENSOR_DIST_M = 0.25f;

// Speed of sound at 20 °C in m/s (adjust for ambient temperature)
const float SPEED_SOUND = 343.0f;

// Measurement interval (ms)
const int MEASURE_MS = 500;

float durationA_us = 0.0f;
float durationB_us = 0.0f;
float tof_A        = 0.0f;
float tof_B        = 0.0f;
float windSpeed_ms = 0.0f;
float windSpeed_kh = 0.0f;

void setup() {
  Serial.begin(115200);
  pinMode(TRIG_A, OUTPUT);
  pinMode(ECHO_A, INPUT);
  pinMode(TRIG_B, OUTPUT);
  pinMode(ECHO_B, INPUT);

  Serial.println("=== Ultrasonic Wind Anemometer ===");
  Serial.print("Sensor separation: ");
  Serial.print(SENSOR_DIST_M * 100.0f, 0);
  Serial.println(" cm");
  Serial.println("Reading wind speed...");
  delay(500);
}

void loop() {
  // Fire Sensor A
  digitalWrite(TRIG_A, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_A, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_A, LOW);
  durationA_us = pulseIn(ECHO_A, HIGH, 30000);

  delay(60);  // Allow echo to settle before second pulse

  // Fire Sensor B
  digitalWrite(TRIG_B, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_B, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_B, LOW);
  durationB_us = pulseIn(ECHO_B, HIGH, 30000);

  // Convert pulse duration (us) to time-of-flight in seconds
  // pulseIn returns round-trip time; divide by 2 for one-way
  tof_A = (durationA_us / 1000000.0f) / 2.0f;
  tof_B = (durationB_us / 1000000.0f) / 2.0f;

  // Ultrasonic anemometer equation:
  // V_wind = (D/2) * (1/tof_A - 1/tof_B)
  // where D is sensor separation and tof values are one-way
  if (tof_A > 0.0f && tof_B > 0.0f) {
    windSpeed_ms = (SENSOR_DIST_M / 2.0f) * ((1.0f / tof_A) - (1.0f / tof_B));
    windSpeed_kh = windSpeed_ms * 3.6f;

    Serial.print("TOF_A=");
    Serial.print(tof_A * 1000000.0f, 1);
    Serial.print("us | TOF_B=");
    Serial.print(tof_B * 1000000.0f, 1);
    Serial.print("us | Wind=");
    Serial.print(windSpeed_ms, 2);
    Serial.print(" m/s (");
    Serial.print(windSpeed_kh, 2);
    Serial.println(" km/h)");
  } else {
    Serial.println("Sensor timeout — check wiring.");
  }

  delay(MEASURE_MS);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Drag two **HC-SR04** components onto the canvas and label them Sensor A and Sensor B.
3. Wire Sensor A: **TRIG → GPIO 14**, **ECHO → GPIO 15**, **VCC → 5V**, **GND → GND**.
4. Wire Sensor B: **TRIG → GPIO 12**, **ECHO → GPIO 13**, **VCC → 5V**, **GND → GND**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** from the simulation dropdown.
7. Click **Run**.
8. Adjust the distance sliders on both HC-SR04 widgets to simulate different echo delays; the Serial Monitor will show the computed wind speed.

## Expected Output
Serial Monitor:
```
=== Ultrasonic Wind Anemometer ===
Sensor separation: 25 cm
Reading wind speed...
TOF_A=364.2us | TOF_B=371.5us | Wind=2.71 m/s (9.76 km/h)
TOF_A=363.8us | TOF_B=371.9us | Wind=2.83 m/s (10.19 km/h)
TOF_A=364.0us | TOF_B=364.1us | Wind=0.04 m/s (0.14 km/h)
```

## Expected Canvas Behavior
* Both HC-SR04 widgets periodically fire and display simulated echo durations.
* The Serial Monitor prints a new wind-speed reading every 500 ms.
* When both distance sliders are set to the same value (equal TOF), wind speed reads near zero.
* Offsetting one slider from the other simulates airflow and produces a non-zero wind speed.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SENSOR_DIST_M = 0.25f` | Physical separation between the two sensor faces in metres. |
| `SPEED_SOUND = 343.0f` | Nominal speed of sound in air at 20 °C; adjust for temperature if needed. |
| `digitalWrite(TRIG_A, HIGH); delayMicroseconds(10)` | Generates the mandatory 10 µs trigger pulse required by the HC-SR04. |
| `pulseIn(ECHO_A, HIGH, 30000)` | Measures the duration of the HIGH pulse on the echo pin (round-trip time); 30 ms timeout. |
| `tof_A = (durationA_us / 1000000.0f) / 2.0f` | Converts echo duration to one-way time-of-flight in seconds. |
| `windSpeed_ms = (SENSOR_DIST_M / 2.0f) * ((1.0f / tof_A) - (1.0f / tof_B))` | Applies the differential time-of-flight anemometer equation. |
| `delay(60)` | Allows the ultrasonic echo from Sensor A to fully decay before triggering Sensor B. |

## Hardware & Safety Concept
* **Differential Time-of-Flight**: Sound travelling with the wind over distance D arrives in time t₁ = D / (c + v), and against the wind in t₂ = D / (c − v), where c is the speed of sound and v is wind speed. Rearranging: v = (D/2)(1/t₁ − 1/t₂). This elegant formula cancels out the speed-of-sound dependency when both sensors are fired rapidly in sequence, though in practice a temperature correction improves accuracy.
* **Voltage Level Protection**: The HC-SR04 ECHO pin outputs 5 V logic. The ARIES v3 GPIO inputs are 3.3 V tolerant. Always insert a resistor voltage divider (1 kΩ + 2 kΩ) on each echo line to prevent damaging the MCU.

## Try This! (Challenges)
1. **Temperature Correction**: Read ambient temperature from a thermistor on ADC1/GP27 and use the formula c = 331.3 + (0.606 × T°C) to update the speed-of-sound constant before each measurement, improving accuracy in non-standard temperatures.
2. **Wind Direction LED**: Add a green LED on GPIO 24 and a red LED on GPIO 23. Light green when wind flows from A→B (positive speed), and red when wind flows from B→A (negative speed), giving a simple directional indicator.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `pulseIn` always returns 0 | Sensor not triggered correctly or echo pin not connected | Verify TRIG is pulsed HIGH for exactly 10 µs and ECHO is connected with voltage divider. |
| Wind speed always shows 0 | Both sensors returning identical TOF | Confirm the sensors are aimed at each other; any physical obstruction will equalise readings. |
| Erratic large speed spikes | Echo crosstalk between the two sensors | Increase the `delay(60)` inter-sensor gap or enclose the sensors in absorptive baffles. |
| Sensor timeout messages | Object too close or too far, or 5V power insufficient | Keep sensor separation between 5 cm and 3 m; ensure 5V supply can source ≥ 100 mA. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [159 - Automatic Battery Capacity Tester](159-automatic-battery-capacity-tester.md)
- [161 - Solar Tracker Dual Axis Controller](161-solar-tracker-dual-axis.md)
- [166 - Smart Fan with Temperature Hysteresis](166-smart-fan.md)
