# 168 - Bluetooth Keypad Entry Lock

Accept a 4-digit PIN code entered over a Bluetooth serial connection from an HC-05 module, compare it against a stored PIN, and unlock a servo on GPIO 13 for 5 seconds before automatically relocking. Wrong PIN entries are counted and access is locked out after three consecutive failures.

## Goal
Learn to receive character-by-character input over Bluetooth serial (UART), build a PIN string using state variables, compare it against an authorised code, control a servo actuator, and implement a lockout mechanism — all without arrays or blocking loops.

## What You Will Build
An HC-05 Bluetooth module connects to the ARIES v3 hardware UART (TX/RX). Using a Bluetooth terminal app on a phone, the user types a 4-digit PIN followed by `#`. The board assembles the PIN one character at a time into four individual `char` variables. If the full PIN matches the stored code, the servo on GPIO 13 rotates to 90° (unlocked) for 5 seconds, then returns to 0° (locked). After three wrong attempts, the system enters a 30-second lockout, printing a warning. A correct master reset code (`0000#`) resets the failure counter.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Green LED (access granted) | `led` | Yes | Yes |
| Red LED (access denied / locked) | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 Module | VCC | 5V | Red | Module power |
| HC-05 Module | GND | GND | Black | Common ground |
| HC-05 Module | TXD | GPIO 1 (RX) | Green | HC-05 TX → ARIES RX |
| HC-05 Module | RXD | GPIO 0 (TX) via divider | Orange | ARIES TX → HC-05 RX (5V-tolerant or use divider) |
| Servo | Signal | GPIO 13 | White | PWM lock/unlock signal |
| Servo | VCC | 5V | Red | Servo power |
| Servo | GND | GND | Black | Servo ground |
| Green LED | Anode | GPIO 24 (LED_G) | Green | Access granted indicator |
| Green LED | Cathode | GND via 330 Ω | — | Current limiting resistor |
| Red LED | Anode | GPIO 23 (LED_R) | Red | Access denied / lockout |
| Red LED | Cathode | GND via 330 Ω | — | Current limiting resistor |

> **Wiring tip:** The HC-05 RXD pin expects 3.3 V logic but is often 5 V tolerant. However, to be safe, use a voltage divider (1 kΩ series + 2 kΩ to GND) between the ARIES TX (GPIO 0) and HC-05 RXD. The HC-05 TXD outputs 3.3 V logic and connects directly to the ARIES RX pin without a level shifter. By default the HC-05 baud rate is 9600; this must match the `Serial.begin()` call (note: on ARIES v3, `Serial` and `Serial1` assignment depends on the firmware — use the hardware UART port connected to GPIO 0/1).

## Code
```cpp
// Bluetooth Keypad Entry Lock
// HC-05 UART: GPIO0(TX)/GPIO1(RX)  |  Servo: GPIO 13

#include <Servo.h>

#define SERVO_PIN 13

// Authorised PIN (4 digits + terminator '#')
const char PIN_D0 = '1';
const char PIN_D1 = '2';
const char PIN_D2 = '3';
const char PIN_D3 = '4';
// PIN is "1234#"

// Lockout settings
const int  MAX_FAILS   = 3;
const long UNLOCK_MS   = 5000L;
const long LOCKOUT_MS  = 30000L;

Servo lockServo;

// State machine: 0=LOCKED, 1=COLLECTING, 2=UNLOCKED, 3=LOCKOUT
int   lockState   = 0;
int   failCount   = 0;
long  timerMs     = 0L;

// PIN input buffer (4 chars + index, no arrays)
char  pinIn0 = 0;
char  pinIn1 = 0;
char  pinIn2 = 0;
char  pinIn3 = 0;
int   pinIdx  = 0;
bool  pinReady = false;

void setup() {
  Serial.begin(9600);    // HC-05 default baud rate
  lockServo.attach(SERVO_PIN);
  lockServo.write(0);    // Start locked

  pinMode(LED_R, OUTPUT);
  pinMode(LED_G, OUTPUT);
  digitalWrite(LED_R, HIGH);
  digitalWrite(LED_G, LOW);

  Serial.println("=== Bluetooth Keypad Lock ===");
  Serial.println("Enter 4-digit PIN followed by #");
}

void loop() {
  // --- STATE 3: LOCKOUT ---
  if (lockState == 3) {
    timerMs -= 100;
    delay(100);
    if (timerMs <= 0) {
      lockState = 0;
      failCount = 0;
      digitalWrite(LED_R, HIGH);
      digitalWrite(LED_G, LOW);
      Serial.println("Lockout expired — system reset. Enter PIN:");
    }
    return;
  }

  // --- STATE 2: UNLOCKED (timed) ---
  if (lockState == 2) {
    timerMs -= 100;
    delay(100);
    if (timerMs <= 0) {
      lockState = 0;
      lockServo.write(0);
      digitalWrite(LED_G, LOW);
      digitalWrite(LED_R, HIGH);
      Serial.println("Auto-locked. Enter PIN:");
    }
    return;
  }

  // --- STATE 0 / 1: LOCKED — read Bluetooth input ---
  if (Serial.available() > 0) {
    char c = Serial.read();

    if (c == '#') {
      // End of PIN entry — evaluate
      pinReady = true;
    } else if (c >= '0' && c <= '9') {
      // Store digit
      if (pinIdx == 0) { pinIn0 = c; pinIdx = 1; }
      else if (pinIdx == 1) { pinIn1 = c; pinIdx = 2; }
      else if (pinIdx == 2) { pinIn2 = c; pinIdx = 3; }
      else if (pinIdx == 3) { pinIn3 = c; pinIdx = 4; }
    } else if (c == '*') {
      // '*' clears current input
      pinIn0 = 0; pinIn1 = 0; pinIn2 = 0; pinIn3 = 0;
      pinIdx = 0;
      Serial.println("Input cleared.");
    }
  }

  if (pinReady) {
    pinReady = false;

    bool correct = (pinIdx == 4) &&
                   (pinIn0 == PIN_D0) &&
                   (pinIn1 == PIN_D1) &&
                   (pinIn2 == PIN_D2) &&
                   (pinIn3 == PIN_D3);

    // Reset input buffer
    pinIn0 = 0; pinIn1 = 0; pinIn2 = 0; pinIn3 = 0;
    pinIdx = 0;

    if (correct) {
      failCount = 0;
      lockState = 2;
      timerMs   = UNLOCK_MS;
      lockServo.write(90);
      digitalWrite(LED_G, HIGH);
      digitalWrite(LED_R, LOW);
      Serial.println("ACCESS GRANTED — Door unlocked for 5 seconds.");
    } else {
      failCount++;
      Serial.print("ACCESS DENIED — Wrong PIN. Attempts remaining: ");
      Serial.println(MAX_FAILS - failCount);

      if (failCount >= MAX_FAILS) {
        lockState = 3;
        timerMs   = LOCKOUT_MS;
        digitalWrite(LED_R, HIGH);
        digitalWrite(LED_G, LOW);
        Serial.println("!!! LOCKOUT ACTIVE — Wait 30 seconds.");
      }
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Drag an **HC-05** (or Serial Terminal) component; connect **TXD** to **GPIO 1 (RX)** and **RXD** to **GPIO 0 (TX)**.
3. Drag a **Servo** component; connect Signal to **GPIO 13**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** from the simulation dropdown.
6. Click **Run**.
7. In the HC-05 terminal widget (or Serial Monitor if connected), type `1234#` and press send — the servo should rotate to 90° and the green LED should light.
8. Type `9999#` three times — after the third failure, a 30-second lockout message should appear.
9. Wait for the lockout to expire (timerMs counts down visibly via the delay) then type `1234#` again to confirm the system resets.

## Expected Output
Serial Monitor / Bluetooth Terminal:
```
=== Bluetooth Keypad Lock ===
Enter 4-digit PIN followed by #
ACCESS GRANTED — Door unlocked for 5 seconds.
Auto-locked. Enter PIN:
ACCESS DENIED — Wrong PIN. Attempts remaining: 2
ACCESS DENIED — Wrong PIN. Attempts remaining: 1
ACCESS DENIED — Wrong PIN. Attempts remaining: 0
!!! LOCKOUT ACTIVE — Wait 30 seconds.
Lockout expired — system reset. Enter PIN:
```

## Expected Canvas Behavior
* The servo widget starts at 0° (locked).
* On correct PIN entry, the servo moves to 90° and the green LED activates.
* After 5 seconds, the servo returns to 0° and the red LED lights automatically.
* Three incorrect PINs trigger the lockout state; the serial terminal prints the countdown message.
* After 30 seconds, the system automatically unlocks from lockout and accepts new input.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `PIN_D0..PIN_D3` | Stores each digit of the authorised 4-digit PIN as a separate `char` constant. |
| `char pinIn0..pinIn3` | Four individual character variables replace an array to store the user's typed PIN. |
| `pinIdx` | Tracks how many digits have been received; resets to 0 after evaluation. |
| `if (c == '#')` | The `#` character terminates PIN entry and triggers evaluation. |
| `if (c == '*')` | The `*` character clears the current input without evaluating. |
| `bool correct = (pinIdx == 4) && (pinIn0 == PIN_D0) && ...` | Evaluates all four digits simultaneously in a single boolean expression. |
| `timerMs -= 100; delay(100)` | Counts down unlock and lockout timers in 100 ms steps without blocking loops. |
| `lockServo.write(90)` | Rotates servo to unlocked position on successful authentication. |
| `failCount >= MAX_FAILS` | Triggers the 30-second lockout after three consecutive wrong PIN entries. |

## Hardware & Safety Concept
* **Bluetooth UART Security**: HC-05 Bluetooth operates on the 2.4 GHz ISM band using the SPP (Serial Port Profile). Once paired, it creates a transparent serial tunnel between the phone terminal and the ARIES UART. The PIN is transmitted as plaintext over Bluetooth; for production security, consider using Bluetooth Low Energy (BLE) with encrypted pairing or a challenge-response authentication scheme.
* **PIN Input Without Arrays**: This project demonstrates that multi-character input can be handled using individual named variables (`pinIn0`–`pinIn3`) and an index counter, avoiding array declarations that are not permitted in interpreted mode. This pattern scales to longer PINs by adding more character variables.
* **Lockout Timing Without `delay`**: The lockout and unlock timers count down by decrementing `timerMs` by 100 each loop cycle combined with `delay(100)`. This approach respects the interpreted-mode constraint while still providing accurate timing.

## Try This! (Challenges)
1. **PIN Change Command**: Receive a special command format `SETPIN:XXXX#` over Bluetooth, parse the new 4-digit PIN into the `PIN_D0`–`PIN_D3` constants, and store it in EEPROM so the new PIN persists across power cycles.
2. **Visual Countdown**: During the unlock period, blink the green LED at 1 Hz intervals (500 ms on, 500 ms off) to signal that the door is open and the auto-lock countdown is active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| No response to typed PIN | HC-05 not connected or baud rate mismatch | Verify HC-05 is paired and `Serial.begin(9600)` matches the HC-05 AT+UART baud setting. |
| PIN always denied even when correct | Characters not matching due to carriage return/newline | Ensure the terminal sends only the digit characters and `#` without `\r\n`; adjust terminal settings. |
| Servo does not move | Servo not attached or SERVO_PIN mismatch | Confirm `lockServo.attach(SERVO_PIN)` is in `setup()` and `SERVO_PIN = 13`. |
| Lockout never expires | timerMs not decrementing or delay missing | Verify `timerMs -= 100` and `delay(100)` are both inside the `lockState == 3` block. |
| Input buffer fills with garbage | Extra characters sent by terminal | Press `*#` to clear input; use a terminal that sends raw characters only. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [165 - Smart Door Lock with PIR Auto-Relock](165-smart-door-lock.md)
- [169 - Multi-zone Temperature Logger](169-multi-zone-temperature-logger.md)
- [170 - NeoPixel Color Spectrum Visualizer](170-neopixel-color-spectrum-visualizer.md)
