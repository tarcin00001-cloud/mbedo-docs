# 36 - ESP32 PIR Motion Sensor Alert

Detect human presence using a PIR (Passive Infrared) motion sensor and trigger a visual and serial alert when movement is detected.

## Goal
Learn how a PIR sensor works, how to read its digital output on the ESP32, and how to handle the sensor's built-in warm-up delay and retrigger timing.

## What You Will Build
A HC-SR501 PIR sensor connected to GPIO 4. When motion is detected the sensor drives GPIO 4 HIGH; the ESP32 lights an LED on GPIO 5 and logs the event. After motion stops, the sensor returns LOW after its output hold time elapses.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-SR501 PIR Sensor Module | `pir_sensor` | Yes | Yes |
| LED (red) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR501 PIR | VCC | 5V (or Vin) | Red | PIR requires 5 V supply (not 3.3 V) |
| HC-SR501 PIR | GND | GND | Black | Common ground |
| HC-SR501 PIR | OUT | GPIO4 | Yellow | Digital output — HIGH on motion (3.3 V safe) |
| LED (red) | Anode (+) | GPIO5 via 330 Ω | Orange | Motion alert LED |
| LED (red) | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** The HC-SR501 requires a **5 V supply** (use the ESP32's Vin or 5 V pin) but its digital output is 3.3 V compatible — safe to connect directly to a GPIO input. Allow **60 seconds of warm-up time** after powering on before expecting reliable motion detection. Two trimmer potentiometers on the module adjust **sensitivity** (detection range up to ~7 m) and **output hold time** (how long OUT stays HIGH after motion stops, from ~3 s to ~5 min). Set the jumper to H (Repeat Trigger) for continuous HIGH while motion persists.

## Code
```cpp
// PIR Motion Sensor Alert
const int PIR_PIN = 4;
const int LED_PIN = 5;

void setup() {
  pinMode(PIR_PIN, INPUT);   // PIR module drives this pin actively
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Serial.begin(115200);
  Serial.println("PIR Motion Sensor — warming up (60 s)...");
  delay(60000);   // PIR sensor warm-up period
  Serial.println("Ready — monitoring for motion.");
}

void loop() {
  int motion = digitalRead(PIR_PIN);

  if (motion == HIGH) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("!! MOTION DETECTED !!");
  } else {
    digitalWrite(LED_PIN, LOW);
    Serial.println("No motion.");
  }

  delay(500);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **PIR Sensor**, and **LED** onto the canvas.
2. Connect PIR **OUT** to **GPIO4**, LED **anode** to **GPIO5**.
3. Paste the code and click **Run**.
4. In MbedO the warm-up delay is simulated — click the PIR widget to trigger a motion event.

## Expected Output
Serial Monitor:
```
PIR Motion Sensor — warming up (60 s)...
Ready — monitoring for motion.
No motion.
No motion.
!! MOTION DETECTED !!
!! MOTION DETECTED !!
No motion.
```

## Expected Canvas Behavior
* LED is off at rest.
* Clicking the PIR sensor widget immediately lights the LED.
* LED turns off when the PIR widget is released (after the hold time expires).

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `delay(60000)` | 60-second warm-up pause — PIR pyroelectric detector requires time to stabilise before giving reliable output. |
| `int motion = digitalRead(PIR_PIN)` | Reads the PIR OUT pin: HIGH during motion, LOW when idle. |
| `if (motion == HIGH)` | Simple level-based detection — the PIR module handles all filtering and hold timing internally. |
| `delay(500)` | 2 Hz polling rate — sufficient because the PIR output is held HIGH for several seconds per detection. |

## Hardware & Safety Concept: Passive Infrared Motion Detection
A PIR sensor detects **changes** in infrared radiation (heat) emitted by warm bodies. It contains two pyroelectric sensing elements wired in opposition. When a warm body moves across the sensor's field of view, it causes one element to receive more radiation than the other, generating a differential signal. The HC-SR501 module uses a BISS0001 processing chip that amplifies this signal, applies a threshold, and drives the OUT pin HIGH for a programmed hold duration. Key characteristics: **passive** (emits no signal of its own — uses ambient IR), **detecting movement only** (a stationary person in the field of view is invisible after several seconds), **Fresnel lens** (the white dome focuses IR onto the sensors and creates the characteristic detection zone pattern). PIR sensors are universal in building security, automatic lighting, and energy-saving systems.

## Try This! (Challenges)
1. **Motion counter**: Count total motion detection events and print the count hourly.
2. **First-detection latch**: Latch the LED on at first motion and require a button press to clear it.
3. **Absence timer**: If no motion is detected for 5 minutes, print "ZONE CLEAR — all quiet."

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OUT always HIGH after warm-up | Sensitivity too high or light shining on sensor | Reduce the sensitivity trimmer; shield the sensor from direct heat sources and sunlight |
| Never detects motion | Supply is 3.3 V instead of 5 V | Move VCC wire to the 5 V (Vin) pin on the ESP32 |
| Output stays HIGH long after motion stops | Hold time trimmer at maximum | Rotate the time trimmer counter-clockwise to reduce the hold time |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [34 - ESP32 LDR Automatic Dark Detector](34-esp32-ldr-automatic-dark-detector.md)
- [37 - ESP32 IR Obstacle Sensor Print](37-esp32-ir-obstacle-sensor-print.md)
- [29 - ESP32 Vibration Sensor Latch Alarm](29-esp32-vibration-sensor-latch-alarm.md)
