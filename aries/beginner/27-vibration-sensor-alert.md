# 27 - Vibration sensor alert

Detect mechanical shock or vibrations using an SW-420 sensor module and trigger a timed LED warning on the VEGA ARIES v3 board.

## Goal
Learn how to interface an SW-420 vibration sensor module, read digital state pulses, and write non-blocking latching code (`millis()`) to keep an alert indicator active for a fixed duration.

## What You Will Build
An SW-420 vibration sensor module is wired to `GPIO 17`. An external warning LED is connected to `GPIO 15`. When a vibration or shock is detected, the sensor module outputs a digital HIGH signal. The VEGA ARIES v3 board detects this pulse, immediately turns ON the warning LED, and sets a non-blocking 1-second timer to keep the LED lit, allowing the brief vibration to be easily seen.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| SW-420 Vibration Sensor | `sw420` (or generic digital input) | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SW-420 Sensor | VCC | 3.3V | Red | Power connection |
| SW-420 Sensor | GND | GND | Black | Ground connection |
| SW-420 Sensor | DO (Digital Out) | GPIO 17 | Green | Digital output signal |
| LED | Anode (+) | GPIO 15 | Red | Digital control signal (through resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The SW-420 module requires active power. Connect VCC to 3.3V and GND to GND. The DO (Digital Output) pin is connected to GPIO 17 on the ARIES board.

## Code
```cpp
// Vibration Sensor Alert - VEGA ARIES v3
const int VIBRATION_PIN = 17;
const int LED_PIN = 15;

unsigned long alertEndTime = 0; // Timestamp when the LED alert should turn off

void setup() {
  // Configure vibration sensor digital pin as input
  pinMode(VIBRATION_PIN, INPUT);
  
  // Configure LED pin as output
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int sensorState = digitalRead(VIBRATION_PIN);
  unsigned long currentTime = millis();
  
  // SW-420 module outputs HIGH when vibration/shock is detected
  if (sensorState == HIGH) {
    digitalWrite(LED_PIN, HIGH);       // Turn ON the warning LED
    alertEndTime = currentTime + 1000; // Set alert duration for 1000 ms (1 second)
  } else {
    // If the alert window has elapsed, turn off the LED
    if (currentTime >= alertEndTime) {
      digitalWrite(LED_PIN, LOW);
    }
  }
  
  delay(10); // High polling rate for shock detection
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, an **SW-420 Vibration Sensor** widget, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire the SW-420's **VCC** to **3.3V**, **GND** to **GND**, and **DO** pin to **GPIO 17**.
3. Wire the **220 Ω Resistor** between **GPIO 15** and the LED's anode (+). Connect the LED's cathode (-) to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Click the SW-420 sensor widget on the canvas to trigger a mock vibration.
7. Observe the external LED immediately turning ON and remaining lit for exactly 1 second before turning OFF.

## Expected Output
Serial Monitor:
```
System Initialized.
Vibration Detected! LED Alert ON
Vibration Cleared. LED Alert OFF
```

## Expected Canvas Behavior
* Tapping/activating the vibration sensor causes the external LED to glow for 1 second, then shut off automatically.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(VIBRATION_PIN, INPUT)` | Configures GPIO 17 as a standard digital input to read output from the comparator IC. |
| `sensorState == HIGH` | Detects a HIGH pulse indicating that the sensor contacts bounced open due to shock. |
| `alertEndTime = currentTime + 1000` | Saves the future time (current time + 1 second) at which the LED should be turned off. |
| `currentTime >= alertEndTime` | Non-blockingly checks if 1 second has elapsed since the last detected vibration. |

## Hardware & Safety Concept
* **SW-420 Sensor Mechanism**: The SW-420 consists of a metal tube containing a small spring element. Under steady conditions, the spring touches a central contact, forming a closed circuit. When vibration occurs, the spring moves and breaks the connection. An LM393 comparator chip on the module converts this intermittent contact bounce into a clean, digital HIGH/LOW pulse.
* **Non-blocking Latching**: Mechanical shock pulses can last less than 5 milliseconds. A human eye cannot see an LED flash for 5 ms, nor can the board run sluggishly if we used a simple `delay(1000)`. Non-blocking latching allows us to extend the LED warning time without freezing the controller's main processing loop.

## Try This! (Challenges)
1. **Intensity Pulse Counter**: Count the number of vibration pulses detected within a 5-second window. If more than 10 pulses occur, sound a buzzer on GPIO 14 (representing heavy machinery fault).
2. **Variable Alert Duration**: Connect a potentiometer on pin `ADC0` and map its value to set the LED alert duration between 500 ms and 5000 ms.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays ON continuously | Sensitivity potentiometer too low | Turn the small blue potentiometer screw on the SW-420 module to adjust the threshold sensitivity |
| LED never turns ON | Incorrect wiring or loose contacts | Ensure the module has power (VCC to 3.3V, GND to GND) and DO is routed to GPIO 17 |
| Warning light turns OFF instantly | Non-blocking timer error | Check that `alertEndTime` is computed using unsigned long arithmetic |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [26 - Tilt sensor alarm](26-tilt-sensor-alarm.md)
- [28 - Laser receiver sensor trigger](28-laser-receiver-sensor-trigger.md) (Next project)
