# 42 - ESP32 Sound Sensor Clap Detector

Use a microphone sound sensor module to detect a clap and toggle an LED — a classic clap-switch implementation.

## Goal
Learn how to read a sound sensor's digital output, detect the brief pulse produced by a sharp clap, and implement a toggle so each clap flips the LED state.

## What You Will Build
A KY-038 sound sensor module on GPIO 4. Each time a loud clap is detected (DO goes HIGH briefly), the LED on GPIO 5 toggles between on and off — a gesture-controlled lamp.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| KY-038 Sound Sensor Module | `sound_sensor` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| KY-038 Sound Sensor | VCC | 3V3 | Red | Module supply voltage |
| KY-038 Sound Sensor | GND | GND | Black | Module ground |
| KY-038 Sound Sensor | DO (digital out) | GPIO4 | Yellow | High pulse on clap |
| LED | Anode (+) | GPIO5 via 330 Ω | Orange | Clap-toggle lamp |
| LED | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** The KY-038 has a sensitivity trimmer — adjust it so the LED on the module board flashes only on a sharp clap, not on normal conversation. Too sensitive causes ambient noise to trigger it constantly. The DO pin produces a brief HIGH pulse (1–10 ms) on a qualifying sound event. Detecting the rising edge of this pulse (rather than the level) ensures each clap counts exactly once.

## Code
```cpp
// Sound Sensor Clap Detector — toggle LED on each clap
const int SOUND_PIN = 4;
const int LED_PIN   = 5;

bool ledState   = false;
bool lastDetect = false;

void setup() {
  pinMode(SOUND_PIN, INPUT);
  pinMode(LED_PIN,   OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Serial.begin(115200);
  Serial.println("Clap Detector ready — clap to toggle lamp.");
}

void loop() {
  bool detected = (digitalRead(SOUND_PIN) == HIGH);

  // Rising edge: sound just started — toggle on first detection
  if (detected && !lastDetect) {
    ledState = !ledState;
    digitalWrite(LED_PIN, ledState);
    Serial.print("CLAP! Lamp is now: ");
    Serial.println(ledState ? "ON" : "OFF");
  }

  lastDetect = detected;
  delay(20);   // 50 Hz polling — fast enough to catch a ~10 ms clap pulse
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Sound Sensor**, and **LED** onto the canvas.
2. Connect Sound Sensor **DO** to **GPIO4**, LED **anode** to **GPIO5**.
3. Paste the code and click **Run**.
4. Click the sound sensor widget to simulate a clap — the LED toggles each click.

## Expected Output
Serial Monitor:
```
Clap Detector ready — clap to toggle lamp.
CLAP! Lamp is now: ON
CLAP! Lamp is now: OFF
CLAP! Lamp is now: ON
```

## Expected Canvas Behavior
* LED starts off.
* Each click of the sound sensor widget flips the LED — on, then off, then on again.
* No output printed between claps.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `bool detected = (digitalRead(SOUND_PIN) == HIGH)` | True while the module's DO pin is HIGH (during a loud sound pulse). |
| `if (detected && !lastDetect)` | Rising-edge detection — only fires once per clap regardless of pulse duration. |
| `ledState = !ledState` | Inverts the boolean lamp state on each rising edge. |
| `delay(20)` | 50 Hz polling — fast enough to reliably catch a 10 ms DO pulse. |

## Hardware & Safety Concept: Electret Microphone Sound Detection
The KY-038 uses an electret condenser microphone connected to an LM393 comparator. The microphone converts sound pressure waves into a tiny AC voltage signal (a few millivolts). This signal is amplified and compared to a reference voltage set by the trimmer potentiometer. When the microphone voltage exceeds the reference, the comparator drives DO HIGH — qualifying the sound as "loud enough". The module is sensitive to **sharp transient sounds** (claps, snaps, knocks) more than continuous noise, because the reference voltage adapts slowly to steady background noise (similar to an automatic gain control). This is why a clap — a sharp impulsive sound — triggers the module even in a noisy room while ordinary conversation may not.

## Try This! (Challenges)
1. **Double-clap**: Require two claps within 800 ms to toggle the LED — single claps are ignored.
2. **Three-command system**: One clap = LED ON, two claps = LED OFF, three claps = blink mode.
3. **Sensitivity logger**: Print the raw ADC value from the AO pin alongside DO — observe the waveform of a clap vs ambient noise.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED toggles constantly (ambient noise) | Sensitivity trimmer too high | Rotate trimmer counter-clockwise until only sharp claps trigger |
| LED never toggles | Sensitivity too low | Rotate trimmer clockwise; clap close to the microphone |
| Multiple toggles per clap | Bounce on the DO pulse | Add a 200 ms lockout: `if (now - lastToggle > 200)` before toggling |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [38 - ESP32 Piezo Vibration Analog Signal Reader](38-esp32-piezo-vibration-analog-signal-reader.md)
- [50 - ESP32 Sound Sensor Threshold Alarm](50-esp32-sound-sensor-threshold-alarm.md)
- [22 - ESP32 Touch Toggle Controller](22-esp32-touch-toggle-controller.md)
