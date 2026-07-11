# 34 - LDR Automatic Dark Detector (Automatic Night Light)

Turn an external LED ON automatically when ambient light levels drop below a set threshold.

## Goal
Learn how to create a threshold trigger using analog values, configure digital output responses, and build a prototype automatic streetlight control system.

## What You Will Build
An LDR is configured in a voltage divider circuit on analog input pin `ADC1` (`GP27`), and an LED is connected to `GPIO 15`. When the light level falls below 40%, the board drives `GPIO 15` HIGH to turn ON the LED. When light rises above 40%, it turns the LED OFF.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| LDR (Photoresistor) | `ldr` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR | Pin 1 | 3.3V | Red | Power connection |
| LDR | Pin 2 | ADC1 (GP27) | Yellow | Analog input signal (at voltage divider midpoint) |
| 10 kΩ Resistor | Pin 1 | ADC1 (GP27) | Yellow | Connected to LDR Pin 2 |
| 10 kΩ Resistor | Pin 2 | GND | Black | Ground connection |
| LED | Anode (+) | GPIO 15 | Red | Output indicator control (via 220 Ω resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Ensure the 220 Ω resistor is in series with the LED to protect the GPIO 15 pin from overcurrent while the LDR voltage divider uses a 10 kΩ resistor to maximize voltage sensitivity.

## Code
```cpp
// LDR Automatic Dark Detector - VEGA ARIES v3
const int LDR_PIN = GP27; // ADC1
const int LED_PIN = 15;   // Warning LED

void setup() {
  // Initialize digital pin 15 as output
  pinMode(LED_PIN, OUTPUT);
  // Initialize serial communication
  Serial.begin(115200);
}

void loop() {
  // Read raw LDR analog value
  int rawValue = analogRead(LDR_PIN);
  
  // Convert 12-bit ADC value (0-4095) to percentage (0-100%)
  int lightPercentage = map(rawValue, 0, 4095, 0, 100);
  
  Serial.print("Light Level: ");
  Serial.print(lightPercentage);
  Serial.print("%");
  
  // Turn LED ON if light percentage is below 40% (Darkness detected)
  if (lightPercentage < 40) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println(" -> Darkness detected! LED ON.");
  } else {
    digitalWrite(LED_PIN, LOW);
    Serial.println(" -> Adequate light. LED OFF.");
  }
  
  // Sample every 500 milliseconds
  delay(500);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, an **LDR**, a **10 kΩ Resistor**, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire the LDR voltage divider to **GP27 (ADC1)**.
3. Wire the LED's anode through the 220 Ω resistor to **GPIO 15** and the cathode to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Adjust the LDR light level slider. Lower the slider to make it "dark" and watch the LED turn on.

## Expected Output
Serial Monitor:
```
System Initialized.
Light Level: 65% -> Adequate light. LED OFF.
Light Level: 28% -> Darkness detected! LED ON.
```

## Expected Canvas Behavior
* The external LED lights up when the LDR slider is moved below 40% intensity, and turns off when moved above 40%.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int LED_PIN = 15;` | Specifies the digital pin driving the night light LED. |
| `map(rawValue, 0, 4095, 0, 100)` | Scales the 12-bit analog input into a 0-100% scale. |
| `if (lightPercentage < 40)` | Conditional check that determines if the ambient light is below the trigger threshold. |

## Hardware & Safety Concept: Threshold Hysteresis and Dark Detection
* **Hysteresis**: In actual sensor systems, readings near a threshold (like exactly 40%) can oscillate rapidly due to electrical noise, turning the output ON and OFF very quickly (chattering). Adding a threshold buffer or hysteresis prevents this.
* **Dark Detection**: Relays or high-power transistors can be used instead of the LED to control large 230V AC streetlights safely using this low-voltage circuit.

## Try This! (Challenges)
1. **Add Hysteresis**: Modify the logic so the LED turns ON when the light level is below 35%, but only turns OFF when the light level rises above 45%. This prevents flicker.
2. **Audible Night Alarm**: Add the active buzzer on `GPIO 14` and trigger it for 500 ms when darkness is first detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays ON permanently | LDR wiring error or incorrect threshold | Check if the LDR and resistor connections are reversed. Swap them or check that your threshold value matches the light range |
| LED stays OFF permanently | Wrong pin configurations | Ensure the LED is wired to GPIO 15 and that the input sensor is on GP27 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [33 - LDR light sensor read](33-ldr-light-sensor-read.md) (Previous project)
- [35 - NTC Thermistor temperature read](35-ntc-thermistor-temperature-read.md) (Next project)
