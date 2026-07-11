# 43 - Flame Sensor Digital Alarm (GPIO Digital Input)

Sound a buzzer alarm automatically when an infrared flame sensor detects a fire source.

## Goal
Understand infrared photodiode flames detectors, configure active-low digital sensor comparator outputs, and program emergency buzzer alert sequences using digital state controls.

## What You Will Build
An Infrared (IR) Flame Sensor is connected to digital input pin `GPIO 17`, and an Active Buzzer is connected to `GPIO 14`. When a fire is detected (pulling `GPIO 17` LOW), the board triggers the buzzer to beep in a rapid emergency pattern.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Flame Sensor Module | `flame_sensor` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Flame Sensor | VCC | 3.3V | Red | Power connection |
| Flame Sensor | DO (Digital Out) | GPIO 17 | Yellow | Digital output (active-low) |
| Flame Sensor | GND | GND | Black | Ground connection |
| Active Buzzer | Positive (+) | GPIO 14 | Red | Alarm output trigger |
| Active Buzzer | Negative (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The digital output (DO) pin on the flame sensor module is active-low. When infrared light in the flame spectrum is detected, the comparator pulls this pin to GND.

## Code
```cpp
// Flame Sensor Digital Alarm - VEGA ARIES v3
const int FLAME_PIN = 17;  // GPIO 17
const int BUZZER_PIN = 14; // GPIO 14

void setup() {
  // Configure the flame sensor pin as input
  pinMode(FLAME_PIN, INPUT);
  
  // Configure the buzzer pin as output
  pinMode(BUZZER_PIN, OUTPUT);
  
  // Initialize serial communication
  Serial.begin(115200);
}

void loop() {
  // Read the digital state of the flame sensor (active-low)
  int flameState = digitalRead(FLAME_PIN);
  
  // Flame detected when sensor output is LOW
  if (flameState == LOW) {
    // sound the buzzer alarm (rapid warning pulse)
    digitalWrite(BUZZER_PIN, HIGH);
    Serial.println("FLAME DETECTED! Buzzer sounding!");
    delay(200);
    
    digitalWrite(BUZZER_PIN, LOW);
    delay(200);
  } else {
    // Keep buzzer silent when no flame is detected
    digitalWrite(BUZZER_PIN, LOW);
  }
  
  // Small loop interval
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **Flame Sensor**, and an **Active Buzzer** onto the canvas.
2. Wire the Flame Sensor's DO to **GPIO 17** and the Buzzer's positive pin to **GPIO 14**.
3. Connect all VCC and GND connections accordingly.
4. Paste the code into the editor.
5. Click **Run**.
6. Slide the simulated flame proximity widget on the canvas to trigger the alarm.

## Expected Output
Serial Monitor:
```
System Initialized.
FLAME DETECTED! Buzzer sounding!
FLAME DETECTED! Buzzer sounding!
```

## Expected Canvas Behavior
* The active buzzer widget triggers and pulses (producing sound animations) whenever the flame sensor slider is set to a "flame present" state.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(FLAME_PIN, INPUT)` | Sets the digital flame sensor pin as an input. |
| `if (flameState == LOW)` | Executes alarm code if the active-low signal indicates a flame has been detected. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives the buzzer high (on) and low (off) with delays to create a pulsing beep. |

## Hardware & Safety Concept: Infrared Photodiodes and Active-Low Logic
* **Infrared Photodiodes**: The flame sensor uses an IR receiver photodiode specifically sensitive to light in the wavelength range of 760 nm to 1100 nm, which matches the spectral emissions of open flames (particularly hydrogen/carbon combustion).
* **Active-Low Logic**: Active-low configurations are common in safety-critical digital sensors. If the sensor loses power or a wire gets cut, the pull-up resistor or state changes can alert the controller.

## Try This! (Challenges)
1. **Visual & Audio Alarm**: Connect the onboard Red LED (`LED_R`) to blink in sync with the buzzer pulses during a flame event.
2. **Flame Safe Lock**: Modify the code to require a push-button press on `GPIO 16` to turn off the alarm once it has been triggered, even if the flame is no longer present.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer is constantly beeping | Flame sensor sensitivity too high or IR interference | Adjust the blue potentiometer on the flame sensor board counterclockwise to decrease sensitivity. Keep the sensor away from direct sunlight |
| Buzzer does not turn on | Reversed polarity of the buzzer | Check that the positive (+) pin of the active buzzer is connected to GPIO 14, and the negative (-) pin is connected to GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [42 - MQ-2 Gas sensor level print](42-mq-2-gas-sensor-level-print.md) (Previous project)
- [44 - Hall Effect magnetic sensor print](44-hall-effect-magnetic-sensor-print.md) (Next project)
