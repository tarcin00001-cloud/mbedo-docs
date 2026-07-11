# 184 - ESP32 DAC Analog Waveform Generator

Build a function generator on the ESP32 that utilizes the internal 8-bit Digital-to-Analog Converter (DAC) peripheral on GPIO 25 to output continuous analog waveforms (Sine, Triangle, Sawtooth, and Square), using a potentiometer to adjust frequency and a button to toggle wave types.

## Goal
Learn how to configure the ESP32's built-in DAC registers, calculate real-time math-based wave coordinates, and generate smooth analog outputs.

## What You Will Build
A potentiometer on GPIO 34 controls the signal frequency. A push button on GPIO 4 cycles through waveform types. The ESP32 calculates output levels and writes them to the DAC pin (GPIO 25), generating true analog voltages (0V to 3.3V) that can be monitored on an oscilloscope.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| 10 kΩ Resistor (for button) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DAC Pin | Analog OUT | GPIO25 | Blue | Waveform output line |
| Potentiometer | Wiper | GPIO34 | Yellow | Frequency control wiper |
| Push Button | Pin 1 / Pin 2 | GPIO4 / 3V3 | Orange / Red | Waveform toggle (active-HIGH) |
| Push Button | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the button with an external 10 kΩ pull-down resistor to GPIO 4. The analog waveform is output on GPIO 25 (DAC Channel 1).

## Code
```cpp
// DAC Analog Waveform Generator (Sine/Triangle/Sawtooth/Square)
#include <Arduino.h>

const int POT_PIN = 34;
const int BUTTON_PIN = 4;
const int DAC_PIN = 25; // DAC1 channel

// Waveform Types
enum WaveType { SINE, TRIANGLE, SAWTOOTH, SQUARE };
WaveType currentWave = SINE;

// Lookup table size for Sine wave (32 points)
const int SINE_POINTS = 32;
uint8_t sineTable[SINE_POINTS];

int stepIndex = 0;
int triangleDir = 1;
int triangleVal = 0;

void generateSineTable() {
  for (int i = 0; i < SINE_POINTS; i++) {
    // Generate sine values scaled to 8-bit range (0 to 255)
    // Angle in radians = i * (2 * pi / points)
    float angle = (float)i * 2.0 * M_PI / (float)SINE_POINTS;
    sineTable[i] = (uint8_t)(127.5 + 127.5 * sin(angle));
  }
}

void setup() {
  Serial.begin(115200);
  
  pinMode(BUTTON_PIN, INPUT); // Pull-down resistor wired
  
  // Initialize DAC on pin 25
  // Sets pin mode and enables DAC peripheral
  pinMode(DAC_PIN, OUTPUT); 
  
  generateSineTable();
  Serial.println("DAC Waveform Generator initialized.");
}

void loop() {
  // 1. Monitor Waveform Toggle Button
  static bool lastBtnState = LOW;
  bool btnState = (digitalRead(BUTTON_PIN) == HIGH);
  
  if (btnState && !lastBtnState) {
    // Cycle to next wave type
    currentWave = (WaveType)((currentWave + 1) % 4);
    
    Serial.print("Switched Waveform to: ");
    switch (currentWave) {
      case SINE:     Serial.println("SINE"); break;
      case TRIANGLE: Serial.println("TRIANGLE"); break;
      case SAWTOOTH: Serial.println("SAWTOOTH"); break;
      case SQUARE:   Serial.println("SQUARE"); break;
    }
    delay(200); // Debounce delay
  }
  lastBtnState = btnState;
  
  // 2. Read Potentiometer to adjust step delay (0 to 10 ms)
  int potRaw = analogRead(POT_PIN);
  int stepDelayUs = map(potRaw, 0, 4095, 50, 10000); // 50 us to 10 ms
  
  // 3. Generate and write analog value to DAC
  uint8_t dacValue = 0;
  
  switch (currentWave) {
    case SINE:
      dacValue = sineTable[stepIndex];
      stepIndex = (stepIndex + 1) % SINE_POINTS;
      break;
      
    case TRIANGLE:
      dacValue = triangleVal;
      triangleVal += (5 * triangleDir); // Step size 5
      if (triangleVal >= 255) {
        triangleVal = 255;
        triangleDir = -1;
      } else if (triangleVal <= 0) {
        triangleVal = 0;
        triangleDir = 1;
      }
      break;
      
    case SAWTOOTH:
      dacValue = stepIndex * 8; // Step size 8
      stepIndex = (stepIndex + 1) % 32;
      break;
      
    case SQUARE:
      if (stepIndex < 16) {
        dacValue = 255;
      } else {
        dacValue = 0;
      }
      stepIndex = (stepIndex + 1) % 32;
      break;
  }
  
  // Write 8-bit value to DAC Channel 1 (GPIO 25)
  // Generates analog voltage: Vout = (dacValue / 255) * 3.3V
  dacWrite(DAC_PIN, dacValue); 
  
  // Delay between steps to set waveform frequency
  delayMicroseconds(stepDelayUs);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer**, and **Button** onto the canvas.
2. Wire Potentiometer to **GPIO34** and Button to **GPIO4**.
3. Paste the code and click **Run**.
4. Adjust the potentiometer. Watch the frequency of the output update on the Serial Monitor.
5. Click the button widget. Cycle through the Sine, Triangle, Sawtooth, and Square wave states.

## Expected Output
Serial Monitor:
```
DAC Waveform Generator initialized.
Switched Waveform to: TRIANGLE
Switched Waveform to: SAWTOOTH
Switched Waveform to: SQUARE
```

DAC Output Voltages (monitored on oscilloscope):
* **Sine Mode**: Smooth sinusoidal curve oscillating between 0V and 3.3V.
* **Triangle Mode**: Ramp wave rising and falling linearly.
* **Square Mode**: Alternates instantly between 0V (LOW) and 3.3V (HIGH).

## Expected Canvas Behavior
* At boot, the DAC starts outputting a Sine wave.
* Adjusting the potentiometer slider alters the step delay, changing the signal frequency.
* Clicking the button widget cycles the waveform type.

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `127.5 + 127.5 * sin(...)` | Calculates sine coordinates scaled to the 8-bit DAC range (0–255). |
| `dacWrite(DAC_PIN, ...)` | Writes the 8-bit value to the DAC registers, generating a true analog voltage on GPIO 25. |
| `delayMicroseconds(...)` | Sets the delay between steps to adjust the waveform frequency. |

## Hardware & Safety Concept: Digital-to-Analog Converters (DAC)
Most microcontroller pins can only output digital signals (0V or 3.3V). To generate intermediate voltages, they use Pulse Width Modulation (PWM), which requires external filtering to become smooth. The ESP32 contains **two hardware 8-bit DAC channels**. The DAC converts digital values directly into true analog voltages on the output pin, allowing generating clean audio signals or reference voltages.

## Try This! (Challenges)
1. **Interactive OLED Oscilloscope**: Add an OLED screen (Project 60) and plot the selected waveform shape graphically.
2. **Audio synthesizer tone**: Connect a speaker (via a 100 Ω current-limiting resistor) to GPIO 25 and generate a musical scale.
3. **High-speed DMA DAC**: Configure the I2S peripheral to stream the waveform table to the DAC in the background using DMA for maximum speed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Waveform frequency is unstable or slow | Serial prints in loop | Remove or comment out debug print statements inside the fast loops |
| DAC output is noisy | Breadboard interference | Add a 0.1 µF bypass capacitor across the DAC output pin and ground |
| Square wave is distorted | High output load | Add a buffer operational amplifier (Op-Amp) to isolate the DAC pin from low-impedance loads |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [181 - ESP32 Custom Hardware Timer Interrupts](181-esp32-custom-hardware-timer-interrupts.md)
- [179 - ESP32 Direct Register Port Manipulation](179-esp32-direct-register-port-manipulation.md)
- [54 - ESP32 DC Motor Speed Scaling](../intermediate/54-esp32-dc-motor-speed-scaling.md)
