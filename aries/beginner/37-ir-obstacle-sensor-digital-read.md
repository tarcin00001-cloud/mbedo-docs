# 37 - IR Obstacle Sensor Digital Read (GPIO Digital Input)

Toggle the state of an external LED when an obstacle is detected by an IR proximity sensor.

## Goal
Understand the operational logic of active infrared reflection sensors, handle active-low digital outputs, and implement edge-triggered actions to control peripheral devices.

## What You Will Build
An Infrared (IR) Obstacle Avoidance sensor is connected to digital input pin `GPIO 17`, and an LED is connected to `GPIO 15`. When an object comes within detection range, the sensor's output pin goes LOW (active-low), triggering the board to toggle the LED ON or OFF.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| IR Obstacle Sensor | `ir_obstacle` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Sensor | VCC | 3.3V | Red | Power connection |
| IR Sensor | OUT | GPIO 17 | Yellow | Digital output (active-low) |
| IR Sensor | GND | GND | Black | Ground connection |
| LED | Anode (+) | GPIO 15 | Red | Output indicator control (via resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The IR obstacle sensor's output pin is typically active-low (logical LOW when an obstacle is detected, HIGH when clear). Ensure you capture this transition properly in code.

## Code
```cpp
// IR Obstacle Sensor Digital Read - VEGA ARIES v3
const int IR_PIN = 17;  // GPIO 17
const int LED_PIN = 15; // Warning LED

// Global states to track sensor transition and output toggle
int lastIrState = HIGH; // Sensor outputs HIGH by default (no obstacle)
int ledState = LOW;

void setup() {
  // Configure the IR pin as digital input
  pinMode(IR_PIN, INPUT);
  // Configure the LED pin as digital output
  pinMode(LED_PIN, OUTPUT);
  
  // Initialize serial communication
  Serial.begin(115200);
}

void loop() {
  // Read the current state of the IR sensor
  int irState = digitalRead(IR_PIN);
  
  // Check if an obstacle was just detected (transition from HIGH to LOW)
  if (irState == LOW && lastIrState == HIGH) {
    // Toggle the LED state
    if (ledState == LOW) {
      ledState = HIGH;
    } else {
      ledState = LOW;
    }
    
    // Write state to physical LED
    digitalWrite(LED_PIN, ledState);
    
    // Print notification
    Serial.print("Obstacle Detected! Toggling LED. LED State: ");
    Serial.println(ledState);
  }
  
  // Save current state for next loop comparison
  lastIrState = irState;
  
  // Polling delay
  delay(50);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, an **IR Obstacle Sensor**, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire the IR sensor to **GPIO 17**, and its power pins to **3.3V** and **GND**.
3. Wire the LED's anode to **GPIO 15** through the resistor and the cathode to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Click the IR obstacle sensor widget on the canvas to trigger an obstacle event.

## Expected Output
Serial Monitor:
```
System Initialized.
Obstacle Detected! Toggling LED. LED State: 1
Obstacle Detected! Toggling LED. LED State: 0
```

## Expected Canvas Behavior
* Placing an obstacle in front of the IR sensor on the canvas causes the warning LED to toggle its state.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int IR_PIN = 17;` | Sets up GPIO 17 as the input pin connected to the IR Obstacle sensor. |
| `if (irState == LOW && lastIrState == HIGH)` | Triggers only when the sensor transitioned from HIGH (clear) to LOW (obstacle present). |
| `digitalWrite(LED_PIN, ledState);` | Drives the digital state (HIGH/LOW) to control the LED. |

## Hardware & Safety Concept: Infrared Reflection and Ambient Light Interference
* **Infrared Reflection**: The sensor consists of an infrared transmitter LED that emits IR light and an IR receiver photodiode that detects reflected IR light. An onboard LM393 voltage comparator compares the receiver voltage to a threshold set by a potentiometer.
* **Interference**: Sunlight or fluorescent lights emit infrared rays that can interfere with the photodiode, causing false triggers. To prevent this, the IR transmitter emits modulated IR signals (usually around 38 kHz), and the receiver is tuned to filter out ambient static IR light.

## Try This! (Challenges)
1. **Proximity Counter**: Increment and print a counter in the Serial Monitor each time an obstacle is detected.
2. **Buzzer Collision Alarm**: Connect a buzzer to `GPIO 14`. Keep the buzzer sounding continuously as long as the obstacle is present (constant LOW state, rather than edge-triggered toggle).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor always reports an obstacle | Sensitivity set too high or direct sunlight | Adjust the onboard sensitivity potentiometer counterclockwise on hardware |
| Sensor does not detect objects | Material is highly absorbing (e.g. black surfaces) | Black surfaces absorb infrared light instead of reflecting it. Use light-colored or reflective obstacles to test the sensor |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [36 - PIR motion sensor trigger](36-pir-motion-sensor-trigger.md) (Previous project)
- [38 - Piezo vibration intensity level print](38-piezo-vibration-intensity-level-print.md) (Next project)
