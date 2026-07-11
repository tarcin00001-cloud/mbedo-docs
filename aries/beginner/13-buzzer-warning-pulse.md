# 13 - Buzzer warning pulse

Create a low-power warning pulse system combining a synchronized buzzer beep and LED flash on the VEGA ARIES v3 board.

## Goal
Learn how to create industrial status indicators that maximize battery life by using brief warning pulses instead of continuous active loads.

## What You Will Build
An active buzzer on `GPIO 14` and a warning LED on `GPIO 15` are programmed to emit a brief 100-millisecond pulse of sound and light once every 2 seconds, indicating that a system is active and in standby.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| LED & 220 Ω Resistor | `led` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Active Buzzer | VCC (+) | GPIO 14 | Blue | Buzzer control signal |
| Active Buzzer | GND (-) | GND | Black | Ground connection |
| Warning LED | Anode (+) | GPIO 15 | Red | LED control signal (via resistor) |
| Warning LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 100 Ω resistor in series with the buzzer and a 220 Ω resistor with the LED to protect the output pins from current spikes.

## Code
```cpp
// Buzzer Warning Pulse - VEGA ARIES v3
const int BUZZER_PIN = 14;
const int LED_PIN = 15;

void setup() {
  // Configure both pins as digital outputs
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  // Fire a brief warning pulse (100 ms duration)
  digitalWrite(BUZZER_PIN, HIGH);
  digitalWrite(LED_PIN, HIGH);
  delay(100);
  
  // Return to low-power standby state (1900 ms duration)
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(LED_PIN, LOW);
  delay(1900);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **Buzzer**, **LED**, and **Resistors** onto the canvas.
2. Wire the Buzzer to **GPIO 14** and the LED anode to **GPIO 15** (through resistor).
3. Connect all ground pins to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Observe the Buzzer and LED emitting brief synchronized pulses every 2 seconds.

## Expected Output
Serial Monitor:
```
System Initialized.
[Standby] Pulse emitted.
[Standby] Pulse emitted.
```

## Expected Canvas Behavior
* Both the buzzer sound waves and the LED flash briefly for 100 milliseconds, go dark/silent for 1.9 seconds, and repeat.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUZZER_PIN, OUTPUT)` | Prepares the digital output pin. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives both pins HIGH simultaneously to initiate the pulse. |
| `delay(1900)` | Keeps the system in a low-power, inactive state between pulses. |

## Hardware & Safety Concept: Duty Cycles and Battery Life
* **Duty Cycles**: The duty cycle of an indicator is the ratio of active time to total period. Here, the system is active for 100 ms out of 2000 ms, giving a duty cycle of `100 / 2000 = 5%`.
* **Battery Life**: By reducing the duty cycle, the average current consumed by the warning system drops by 95%. In battery-powered remote warning stations, this configuration allows the device to run for months instead of days.

## Try This! (Challenges)
1. **Urgent Warning**: Increase the pulse rate by changing the standby delay to 400 milliseconds.
2. **Double Blink pulse**: Program the system to flash the LED twice before firing the buzzer tone.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sound is continuous | Logic logic error | Verify that both pins are set `LOW` in the second half of the loop function |
| LED flashes but no sound | Buzzer wire broken | Check that the buzzer's positive terminal is securely connected to GPIO 14 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [12 - Dual RGB LED Sync](12-dual-rgb-led-sync.md)
- [14 - LED PWM brightness fade (GPIO PWM)](14-led-pwm-brightness-fade.md) (Next project)
