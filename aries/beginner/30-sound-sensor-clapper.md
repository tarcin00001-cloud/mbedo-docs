# 30 - Sound sensor clapper (Clap → LED toggle)

Toggle the state of an external LED by clapping or making a loud sound using a digital sound sensor on the VEGA ARIES v3 board.

## Goal
Learn how to interface digital sound (microphone) sensors, understand comparator outputs, and implement edge detection with a long cooldown delay to filter out audio echo.

## What You Will Build
A digital sound sensor module is connected to `GPIO 17`. An external LED is connected to `GPIO 15`. The sound module contains a microphone and comparator circuit. When a clap is detected, the module outputs a brief digital LOW pulse. The VEGA ARIES v3 board detects this transition (falling edge) and toggles the state of the LED (ON if it was OFF, and vice-versa). A 300 ms cooldown delay prevents the echoing sound wave from double-triggering the switch.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Sound Sensor Module | `sound_sensor` (or generic digital input) | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Sound Sensor | VCC | 3.3V | Red | Power connection |
| Sound Sensor | GND | GND | Black | Ground connection |
| Sound Sensor | OUT (Digital Out) | GPIO 17 | Green | Digital output signal |
| LED | Anode (+) | GPIO 15 | Red | Digital control signal (through resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The digital sound sensor requires 3.3V power. Ensure VCC connects to 3.3V and GND connects to GND. Wire the OUT (or OUT_D / DO) terminal to GPIO 17. The LED is wired to GPIO 15 with a 220 Ω series resistor.

## Code
```cpp
// Sound Sensor Clapper - VEGA ARIES v3
const int SOUND_PIN = 17;
const int LED_PIN = 15;

int lastSoundState = HIGH; // Standard sound modules output HIGH when quiet
int ledState = LOW;        // Track the toggle state of the LED

void setup() {
  // Configure the sound sensor pin as input
  pinMode(SOUND_PIN, INPUT);
  
  // Configure the LED pin as output
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, ledState);
}

void loop() {
  int currentSoundState = digitalRead(SOUND_PIN);
  
  // Detect a falling edge (transition from HIGH to LOW = clap detected)
  if (lastSoundState == HIGH && currentSoundState == LOW) {
    ledState = !ledState;            // Toggle LED state
    digitalWrite(LED_PIN, ledState); // Update LED output
    
    // Cooldown delay is critical to let the sound wave decay and avoid multiple triggers
    delay(300); 
  }
  
  lastSoundState = currentSoundState;
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **Sound Sensor** widget, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire the Sound Sensor's **VCC** to **3.3V**, **GND** to **GND**, and **OUT** to **GPIO 17**.
3. Wire the **220 Ω Resistor** in series between **GPIO 15** and the LED's anode (+). Connect the LED's cathode (-) to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Click the Sound Sensor widget on the canvas to simulate a clap.
7. Observe the external LED toggling ON and OFF with each simulated clap event.

## Expected Output
Serial Monitor:
```
System Initialized.
Loud Sound Detected! Toggling LED state.
GPIO 15 State: HIGH
Loud Sound Detected! Toggling LED state.
GPIO 15 State: LOW
```

## Expected Canvas Behavior
* Clapping (clicking the sound sensor widget) toggles the illumination of the external LED.
* The LED stays in its current state until another loud sound is registered.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(SOUND_PIN, INPUT)` | Configures GPIO 17 as a digital input pin to read signals from the microphone module. |
| `lastSoundState == HIGH && currentSoundState == LOW` | Detects a falling edge, which occurs when a sound exceeds the comparator threshold and drives the output LOW. |
| `ledState = !ledState` | Inverts the current LED state variable. |
| `delay(300)` | Suspends program execution for 300 ms to filter out the reverberation and echoing of the clap. |

## Hardware & Safety Concept
* **Sound Module Comparator**: The module contains an electret microphone capsule and an LM393 comparator integrated circuit. The microphone converts sound waves into a small analog voltage. The onboard potentiometer sets a reference voltage. If the sound signal voltage exceeds the reference voltage, the LM393 comparator outputs a digital LOW pulse.
* **Sensitivity Tuning**: On physical hardware, you must turn the potentiometer screw to calibrate the sensor. If it is too sensitive, it will stay LOW constantly (always triggered). If it is not sensitive enough, it will never trigger.
* **Echo Cooldown**: When you clap, the sound pressure waves bounce off walls, causing multiple peaks over a duration of 50 to 200 milliseconds. Without the `delay(300)` cooldown window, the microcontroller would register these echoes as multiple claps, causing the LED to toggle randomly.

## Try This! (Challenges)
1. **Double Clap Switch**: Implement code (using `millis()`) that only toggles the LED if two distinct claps are heard within a 1-second window.
2. **Audio Peak Indicator**: Wire the onboard Green LED (`LED_G` on GPIO 24) to flash for 100 ms on every sound peak, while the external LED continues to latch.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED toggles constantly or stays ON | Potentiometer incorrectly calibrated | Adjust the module's potentiometer sensitivity using a screwdriver until the onboard sensor indicator LED turns off in a quiet room |
| LED does not respond to sounds | Sensor too insensitive or no power | Verify the module's power LED is glowing red. Turn the sensitivity potentiometer clockwise to increase sensitivity |
| LED flickers during clap | Cooldown delay too short | Increase the `delay(300)` in the loop block to `delay(400)` or `delay(500)` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [29 - Slide switch state reporter](29-slide-switch-state-reporter.md)
- [01 - Onboard RGB LED Blink (Red channel)](01-onboard-rgb-led-blink-red.md) (Restart beginner track)
