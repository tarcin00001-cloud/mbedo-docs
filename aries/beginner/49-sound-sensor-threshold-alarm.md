# 49 - Sound Sensor Threshold Alarm (GPIO Analog pin)

Sound a buzzer warning alarm automatically when a loud noise exceeds a set threshold.

## Goal
Understand the properties of microphone and audio amplifier modules, read analog sound wave amplitudes using the ADC, and build a noise-activated alert switch.

## What You Will Build
An analog sound sensor module (electret microphone breakout) is connected to analog input pin `ADC0` (`GP26`), and an active buzzer is connected to digital output pin `GPIO 14`. When a sound event (such as a clap or shout) exceeds the set threshold level, the buzzer sounds for 200 milliseconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Analog Sound Sensor Module | `sound_sensor` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Sound Sensor | VCC | 3.3V | Red | Power connection |
| Sound Sensor | AO (Analog Out) | ADC0 (GP26) | Yellow | Analog sound level signal |
| Sound Sensor | GND | GND | Black | Ground connection |
| Active Buzzer | Positive (+) | GPIO 14 | Red | Alarm output trigger |
| Active Buzzer | Negative (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The sound sensor features both Analog Output (AO) and Digital Output (DO). Connect the AO pin to GP26 (ADC0) to measure the raw sound amplitude. DO can remain disconnected.

## Code
```cpp
// Sound Sensor Threshold Alarm - VEGA ARIES v3
const int SOUND_PIN = GP26;  // ADC0
const int BUZZER_PIN = 14;   // GPIO 14

// Sound threshold limit (adjustable based on ambient noise)
const int THRESHOLD = 2500;

void setup() {
  // Configure the buzzer pin as output
  pinMode(BUZZER_PIN, OUTPUT);
  // Initialize serial communication
  Serial.begin(115200);
}

void loop() {
  // Read the raw analog sound amplitude (0 to 4095)
  int soundLevel = analogRead(SOUND_PIN);
  
  // If the sound exceeds the threshold, trigger the alarm
  if (soundLevel > THRESHOLD) {
    // Sound the buzzer
    digitalWrite(BUZZER_PIN, HIGH);
    
    Serial.print("LOUD SOUND DETECTED! Level: ");
    Serial.println(soundLevel);
    
    // Hold beep for 200 milliseconds
    delay(200);
    
    // Silence the buzzer
    digitalWrite(BUZZER_PIN, LOW);
  } else {
    // Ensure buzzer is off
    digitalWrite(BUZZER_PIN, LOW);
  }
  
  // Fast sampling rate to capture audio peaks
  delay(10);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **Sound Sensor**, and an **Active Buzzer** onto the canvas.
2. Wire the Sound Sensor's AO to **GP26 (ADC0)**.
3. Wire the Buzzer's positive pin to **GPIO 14** and negative to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Adjust the volume/noise slider on the virtual sound sensor widget to exceed the threshold and verify that the buzzer sounds.

## Expected Output
Serial Monitor:
```
System Initialized.
LOUD SOUND DETECTED! Level: 2850
```

## Expected Canvas Behavior
* The active buzzer widget pulses and emits sound animations whenever the sound sensor slider is dragged past 2500.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int SOUND_PIN = GP26;` | Defines the sound sensor analog input pin. |
| `analogRead(SOUND_PIN)` | Reads the instantaneous amplified sound wave amplitude. |
| `if (soundLevel > THRESHOLD)` | Compares the raw sound amplitude with the target threshold. |

## Hardware & Safety Concept: Electret Condenser Microphones and Audio Amplifiers
* **Electret Condenser Microphones**: The sound sensor uses a small electret capsule containing a thin metallized diaphragm placed near a backplate, forming a capacitor. Sound waves vibrate the diaphragm, changing the capacitance and generating a tiny AC voltage.
* **Audio Amplifiers**: The AC voltage from the microphone capsule is too small (microvolts) for microcontrollers to read. An onboard operational amplifier (like the LM393 or MAX4466) amplifies this signal and shifts the baseline voltage (usually to VCC/2) so the ADC can read the full sound wave.

## Try This! (Challenges)
1. **Clap Switch Toggle**: Modify the code to toggle the state of the onboard Green LED (`LED_G`) each time a loud clap (sound level > 2500) is detected, rather than just sounding a buzzer.
2. **Noise Meter Display**: Light up different onboard LEDs based on noise volume (e.g. Green LED for soft sound, Blue LED for moderate sound, Red LED for loud noise).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer is constantly beeping | Threshold is too low or physical microphone is picking up buzzer noise | Increase the `THRESHOLD` value in code or adjust the physical sensitivity trim pot. Make sure the buzzer is located away from the microphone capsule |
| No response to loud noises | Sensor gain is set too low or wrong output pin | Verify you are connected to the AO (Analog Output) pin of the sensor and that the sensitivity potentiometer on the sensor board is turned clockwise to increase gain |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [48 - LDR ambient light LED dimmer](48-ldr-ambient-light-led-dimmer.md) (Previous project)
- [50 - Analog Joysticks button press](50-analog-joysticks-button-press.md) (Next project)
