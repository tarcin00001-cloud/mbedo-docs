# 14 - ESP32 LED SOS Rescue Beacon

Build an emergency Morse code SOS rescue beacon using an external LED.

## Goal
Learn how to code Morse code timing specifications (dots vs. dashes), implement sequential logic blocks, and build visual signaling warning systems.

## What You Will Build
An external LED connected to GPIO 22 flashes the international Morse code SOS distress signal (three short flashes, three long flashes, three short flashes) repeatedly, with appropriate intervals.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Red or Amber) | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Optional | Yes (current limit) |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (via 220 Ω) | GPIO22 | Red | Signal beacon output |
| LED | Cathode | GND | Black | Ground return |

> **Wiring tip:** Wire the LED anode to GPIO22 via a 220 ohm resistor. The cathode connects directly to GND.

## Code
```cpp
// External LED connected to GPIO 22
const int BEACON_PIN = 22;

// Morse code timing definitions in milliseconds
const int DOT_TIME = 200;      // Short flash (S)
const int DASH_TIME = 600;     // Long flash (O)
const int INNER_LETTER = 200;  // Pause between parts of the same letter
const int WORD_DELAY = 1500;   // Pause between complete SOS cycles

void setup() {
  pinMode(BEACON_PIN, OUTPUT);
  digitalWrite(BEACON_PIN, LOW);
  
  Serial.begin(115200);
  Serial.println("ESP32 SOS Beacon Online.");
}

// Flash a dot (short flash)
void flashDot() {
  digitalWrite(BEACON_PIN, HIGH);
  delay(DOT_TIME);
  digitalWrite(BEACON_PIN, LOW);
  delay(INNER_LETTER);
}

// Flash a dash (long flash)
void flashDash() {
  digitalWrite(BEACON_PIN, HIGH);
  delay(DASH_TIME);
  digitalWrite(BEACON_PIN, LOW);
  delay(INNER_LETTER);
}

void loop() {
  Serial.println(">> Sending SOS Signal <<");
  
  // 1. Send 'S' (three dots: . . .)
  flashDot();
  flashDot();
  flashDot();
  delay(400); // Pause between letters
  
  // 2. Send 'O' (three dashes: - - -)
  flashDash();
  flashDash();
  flashDash();
  delay(400); // Pause between letters
  
  // 3. Send 'S' (three dots: . . .)
  flashDot();
  flashDot();
  flashDot();
  
  // Wait before starting the next transmission
  delay(WORD_DELAY);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Connect LED **A** to **GPIO22**.
3. Connect LED **C** to **GND**.
4. Paste code, select interpreted C++ mode, and click **Run**.
5. Observe the flashing Morse code pattern and check the Serial Monitor.

## Expected Output
Serial Monitor:
```
ESP32 SOS Beacon Online.
>> Sending SOS Signal <<
>> Sending SOS Signal <<
```

## Expected Canvas Behavior
* The LED flashes: 3 quick flashes, 3 long flashes, 3 quick flashes.
* There is a 1.5-second dark period, then the sequence repeats.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `flashDot();` | Calls the helper function to flash the LED for 200 ms. |
| `flashDash();` | Calls the helper function to flash the LED for 600 ms (three times the length of a dot, following standard Morse specifications). |

## Hardware & Safety Concept: Morse Code Timings
Morse code timing is defined relative to the length of a single dot:
- **Dot duration**: 1 unit (e.g. 200 ms)
- **Dash duration**: 3 units (600 ms)
- **Space between elements**: 1 unit (200 ms)
- **Space between letters**: 3 units (600 ms)
- **Space between words**: 7 units (1400 ms)
Replicating these proportions exactly ensures that search and rescue receivers can automatically decode the light flashes using signal processing.

## Try This! (Challenges)
1. **Audible Beacon**: Connect a buzzer on GPIO 25 and pulse it in sync with the LED to create a sound-and-light rescue beacon.
2. **Speed Modulation**: Add a button on GPIO 4 that switches the base dot time between 100 ms (fast transmit) and 300 ms (slow/long range transmit).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Morse code seems irregular or runs together | Missing spacing delays | Ensure that you have `delay(INNER_LETTER)` inside the `flashDot` and `flashDash` functions, which separates consecutive marks. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [02 - ESP32 LED Blink Interval](02-esp32-led-blink-interval.md)
- [06 - ESP32 Active Buzzer Beep Alarm Pattern](06-esp32-active-buzzer-beep-alarm-pattern.md)
