# 04 - ESP32 Multiple LED Chase

Build a sequential LED light bar chaser (often called a "Knight Rider" scan effect) using four external LEDs.

## Goal
Learn how to define, initialize, and control an array of output pins using loops (`for` loops), reducing repetitive code and building dynamic visual patterns.

## What You Will Build
Four external LEDs connected to GPIO pins 18, 19, 21, and 22 light up one after the other in sequence, creating a scanning or chasing light bar effect.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LEDs × 4 | `led` | Yes (four LEDs) | Yes |
| 220 Ω Resistors × 4 | `resistor` | Optional | Yes (current limit) |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED 1 | Anode (via 220 Ω) | GPIO18 | Red | First step in chase |
| LED 2 | Anode (via 220 Ω) | GPIO19 | Orange | Second step in chase |
| LED 3 | Anode (via 220 Ω) | GPIO21 | Yellow | Third step in chase |
| LED 4 | Anode (via 220 Ω) | GPIO22 | Green | Fourth step in chase |
| All Cathodes | Ground | GND | Black | Shared return rail to ground |

> **Wiring tip:** Ground pins on the ESP32 can be shared by using a common breadboard rail. Connect each anode to its respective GPIO pin via a resistor.

## Code
```cpp
// Array of pins connected to the LEDs in physical order
const int ledPins[] = {18, 19, 21, 22};
const int numLeds = 4;

const int CHASE_SPEED = 150; // Speed of the chase in milliseconds

void setup() {
  // Use a loop to configure all pins in the array as OUTPUT
  for (int i = 0; i < numLeds; i++) {
    pinMode(ledPins[i], OUTPUT);
  }
  
  Serial.begin(115200);
  Serial.println("ESP32 LED Chaser Initialised.");
}

void loop() {
  // Scan forward: LED 1 -> 2 -> 3 -> 4
  for (int i = 0; i < numLeds; i++) {
    // Turn the active LED ON
    digitalWrite(ledPins[i], HIGH);
    Serial.print("Active LED Pin: ");
    Serial.println(ledPins[i]);
    delay(CHASE_SPEED);
    
    // Turn the active LED OFF before moving to the next
    digitalWrite(ledPins[i], LOW);
  }
  
  // Scan backward: LED 3 -> 2
  // Creates a seamless back-and-forth bounce
  for (int i = numLeds - 2; i > 0; i--) {
    digitalWrite(ledPins[i], HIGH);
    Serial.print("Active LED Pin: ");
    Serial.println(ledPins[i]);
    delay(CHASE_SPEED);
    digitalWrite(ledPins[i], LOW);
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **four LEDs** onto the canvas.
2. Connect LED anodes to **GPIO18**, **GPIO19**, **GPIO21**, and **GPIO22**.
3. Connect all cathodes to ESP32 **GND**.
4. Paste the code into the editor.
5. Select interpreted mode and click **Run**.

## Expected Output
Serial Monitor:
```
ESP32 LED Chaser Initialised.
Active LED Pin: 18
Active LED Pin: 19
Active LED Pin: 21
Active LED Pin: 22
Active LED Pin: 21
Active LED Pin: 19
```

## Expected Canvas Behavior
* The 4 LEDs flash in sequence back and forth: LED1 -> LED2 -> LED3 -> LED4 -> LED3 -> LED2 -> LED1, repeating.
* The terminal logs the active pin values in sync with the lights.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `const int ledPins[] = {18, 19, 21, 22};` | Declares an array of integer values containing the target GPIO pins. |
| `for (int i = 0; i < numLeds; i++)` | Iterates through the array indices (0, 1, 2, 3) to execute pin configurations. |
| `ledPins[i]` | Accesses the GPIO pin number stored at index `i` of the array. |

## Hardware & Safety Concept: Pin Array Mapping
Using pin arrays is a standard software practice in embedded development. It abstracts hardware connections from logical sequences. If the layout changes (e.g. if the board is routed differently on a PCB), you only edit the initial array definition `ledPins[]` instead of changing dozens of digital writes throughout the code.

## Try This! (Challenges)
1. **Speed Controller**: Add a potentiometer on GPIO 34. Read its analog value and map it to change the `CHASE_SPEED` from 50 ms to 500 ms.
2. **Ping-Pong Split**: Modify the loops so that two LEDs turn on from the outside edge, meeting in the middle, and then split outwards.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| One LED in the middle of the chain does not turn ON | Broken array indexing or wiring | Check that your array pin values match your physical wiring exactly. GPIO 20 is not exposed on DevKitC, which is why the array skips to 21. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [03 - ESP32 Alternate LED Blink](03-esp32-alternate-led-blink.md)
- [10 - ESP32 RGB LED Cycle](10-esp32-rgb-led-cycle.md)
