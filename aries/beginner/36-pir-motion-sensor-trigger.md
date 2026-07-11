# 36 - PIR Motion Sensor Trigger (GPIO Digital Input)

Toggle the state of an external LED whenever motion is detected by a PIR sensor.

## Goal
Learn how to interface a digital input sensor, implement edge-detection (state-change detection) to capture a trigger event, and control outputs based on motion.

## What You Will Build
A Passive Infrared (PIR) motion sensor is connected to digital input pin `GPIO 17`, and an LED is connected to `GPIO 15`. When a person or animal moves in front of the sensor, the output transitions from LOW to HIGH, causing the board to toggle the state of the LED.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| PIR Sensor | VCC | 5V | Red | Power connection (PIR is typically 5V-compatible) |
| PIR Sensor | OUT | GPIO 17 | Yellow | Digital motion detection output |
| PIR Sensor | GND | GND | Black | Ground connection |
| LED | Anode (+) | GPIO 15 | Red | Status indicator (via resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Although the VEGA ARIES v3 operates on 3.3V logic, many PIR sensors require a 5V power supply. However, their signal output pin is usually 3.3V-tolerant, which makes it safe to connect directly to GPIO 17.

## Code
```cpp
// PIR Motion Sensor Trigger - VEGA ARIES v3
const int PIR_PIN = 17; // GPIO 17
const int LED_PIN = 15; // Warning LED

// Variables to keep track of sensor and LED states
int lastPirState = LOW;
int ledState = LOW;

void setup() {
  // Configure pins
  pinMode(PIR_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  
  // Initialize serial communication
  Serial.begin(115200);
}

void loop() {
  // Read current state of the PIR sensor
  int pirState = digitalRead(PIR_PIN);
  
  // Check if PIR state transitioned from LOW to HIGH (motion starting)
  if (pirState == HIGH && lastPirState == LOW) {
    // Toggle the LED state variable
    if (ledState == LOW) {
      ledState = HIGH;
    } else {
      ledState = LOW;
    }
    
    // Write state to the LED
    digitalWrite(LED_PIN, ledState);
    
    // Print the event to the Serial Monitor
    Serial.print("Motion Detected! LED State Toggled to: ");
    Serial.println(ledState);
  }
  
  // Update last state tracker
  lastPirState = pirState;
  
  // Polling rate delay
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **PIR Sensor**, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire the PIR sensor: VCC to **5V**, OUT to **GPIO 17**, and GND to **GND**.
3. Wire the LED's anode through the resistor to **GPIO 15** and the cathode to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Click the PIR sensor widget on the canvas to trigger a motion event.

## Expected Output
Serial Monitor:
```
System Initialized.
Motion Detected! LED State Toggled to: 1
Motion Detected! LED State Toggled to: 0
```

## Expected Canvas Behavior
* Toggling or triggering the PIR motion sensor on the canvas toggles the external LED on and off.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(PIR_PIN, INPUT)` | Configures the PIR sensor pin as a digital input. |
| `pirState == HIGH && lastPirState == LOW` | Checks if a transition from no-motion to motion has just occurred (edge detection). |
| `ledState = !ledState` | Inverts the state of the LED state tracker. |

## Hardware & Safety Concept: Passive Infrared Radiation and Sensor Delays
* **Passive Infrared Radiation (PIR)**: PIR sensors detect movement by measuring changes in the infrared radiation (heat) emitted by surrounding objects. A dual-element pyroelectric sensor inside measures differences in IR levels across two slots.
* **Sensor Delays**: Most physical PIR sensors have a hardware delay potentiometer (usually adjustable from 3 seconds to 5 minutes) and a trigger mode jumper (repeatable vs. non-repeatable). Software must account for these delays, as the pin will remain HIGH for the duration of the hardware timer.

## Try This! (Challenges)
1. **Intruder Alert Buzzer**: Connect an active buzzer to `GPIO 14`. When motion is detected, beep the buzzer rapidly 3 times.
2. **Motion Timeout**: Instead of toggling the LED, modify the code to turn the LED ON when motion is detected, and turn it OFF only after 5 seconds of no motion. (Hint: use `millis()` to track time).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| PIR sensor constantly reports HIGH | High sensitivity setting or warm ambient environment | On hardware, turn the sensitivity potentiometer counterclockwise. In simulator, ensure you are not continuously triggering the widget |
| LED does not toggle | Missing state tracking variable | Ensure you are monitoring the *transition* of the state (`pirState == HIGH && lastPirState == LOW`), rather than just checking if the pin is HIGH |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [35 - NTC Thermistor temperature read](35-ntc-thermistor-temperature-read.md) (Previous project)
- [37 - IR Obstacle sensor digital read](37-ir-obstacle-sensor-digital-read.md) (Next project)
