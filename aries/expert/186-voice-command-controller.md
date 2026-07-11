# 186 - Voice Command Controller

Control multiple outputs (LED, Buzzer, Servo) by receiving single-character voice commands transmitted wirelessly from a smartphone speech-recognition app via an HC-05 Bluetooth module.

## Goal

Learn how to parse incoming serial characters from an HC-05 Bluetooth module and dispatch them to multiple output peripherals, building a character-based command dispatcher without loops or buffers.

## What You Will Build

A smartphone running a Bluetooth serial terminal app (e.g. Serial Bluetooth Terminal) with a connected speech-recognition input sends single-character commands ('L', 'B', 'S', 'O') over Bluetooth to the HC-05 module wired to the ARIES v3 UART pins. The board parses the incoming character in `loop()` and toggles the Red LED, fires the buzzer, or sweeps the servo based on the command received.

## Parts Needed

| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| Red LED + 220 Ω Resistor | `led` | Yes | Yes |
| Jumper Wires | — | No | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 Module | VCC | 5V | Red | Module requires 5 V supply |
| HC-05 Module | GND | GND | Black | Common ground |
| HC-05 Module | TXD | GPIO 0 (RX) | Green | HC-05 TX → ARIES RX |
| HC-05 Module | RXD | GPIO 1 (TX) | Yellow | ARIES TX → HC-05 RX |
| Active Buzzer | + | GPIO 14 | Orange | Buzzer positive |
| Active Buzzer | − | GND | Black | Buzzer negative |
| Servo Motor | Signal | GPIO 13 | Blue | PWM servo signal |
| Servo Motor | VCC | 5V | Red | Servo power |
| Servo Motor | GND | GND | Black | Servo ground |
| Red LED | Anode (+) | GPIO 23 (LED_R) | Red | Onboard LED |

> **Wiring tip:** On real hardware the HC-05 RXD pin must not receive 5 V logic — the ARIES v3 GPIO output is 3.3 V, so no voltage divider is required for the ARIES TX → HC-05 RX line. Use a Bluetooth serial terminal app such as *Serial Bluetooth Terminal* (Android) with voice-input keyboard to send single characters matching the commands below.

## Code

```cpp
// 186 - Voice Command Controller
// HC-05 Bluetooth + Speech Recognition multi-output control
// L = toggle LED, B = beep buzzer, S = sweep servo, O = all off

#define PIN_LED   23
#define PIN_BUZZ  14
#define PIN_SERVO 13

int ledState   = LOW;
int buzzActive = 0;
int buzzTimer  = 0;
int servoAngle = 0;
int servoDir   = 1;
int sweeping   = 0;
int sweepStep  = 0;

void setup() {
  Serial.begin(9600);
  pinMode(PIN_LED,   OUTPUT);
  pinMode(PIN_BUZZ,  OUTPUT);
  pinMode(PIN_SERVO, OUTPUT);
  digitalWrite(PIN_LED,  LOW);
  digitalWrite(PIN_BUZZ, LOW);
  analogWrite(PIN_SERVO, 0);
  Serial.println("Voice Command Controller Ready.");
  Serial.println("L=LED  B=Buzzer  S=Servo  O=All off");
}

void loop() {
  if (Serial.available() > 0) {
    char cmd = (char)Serial.read();

    if (cmd == 'L' || cmd == 'l') {
      ledState = !ledState;
      digitalWrite(PIN_LED, ledState);
      Serial.println(ledState ? "CMD: LED ON" : "CMD: LED OFF");
    }

    if (cmd == 'B' || cmd == 'b') {
      buzzActive = 1;
      buzzTimer  = 0;
      Serial.println("CMD: BUZZER beep");
    }

    if (cmd == 'S' || cmd == 's') {
      sweeping   = 1;
      sweepStep  = 0;
      servoAngle = 0;
      servoDir   = 1;
      Serial.println("CMD: SERVO sweep start");
    }

    if (cmd == 'O' || cmd == 'o') {
      ledState   = LOW;
      buzzActive = 0;
      sweeping   = 0;
      servoAngle = 0;
      digitalWrite(PIN_LED,  LOW);
      digitalWrite(PIN_BUZZ, LOW);
      analogWrite(PIN_SERVO, 0);
      Serial.println("CMD: ALL OFF");
    }
  }

  if (buzzActive) {
    buzzTimer++;
    if (buzzTimer < 25) {
      digitalWrite(PIN_BUZZ, HIGH);
    } else {
      digitalWrite(PIN_BUZZ, LOW);
      buzzActive = 0;
      buzzTimer  = 0;
    }
  }

  if (sweeping) {
    sweepStep++;
    if (sweepStep >= 3) {
      sweepStep  = 0;
      servoAngle += (servoDir * 5);
      if (servoAngle >= 180) {
        servoAngle = 180;
        servoDir   = -1;
      }
      if (servoAngle <= 0) {
        servoAngle = 0;
        sweeping   = 0;
        Serial.println("Servo sweep complete.");
      }
      int pwmVal = (int)((servoAngle / 180.0) * 255);
      analogWrite(PIN_SERVO, pwmVal);
    }
  }

  delay(20);
}
```

## What to Click in MbedO

1. Drag **VEGA ARIES v3**, **HC-05**, **Active Buzzer**, **Servo (SG90)**, and a **Red LED** onto the canvas.
2. Wire the HC-05: **TXD → GPIO 0**, **RXD → GPIO 1**, **VCC → 5V**, **GND → GND**.
3. Wire the Buzzer **+** to **GPIO 14** and **−** to **GND**.
4. Wire the Servo **Signal** to **GPIO 13**, **VCC** to **5V**, **GND** to **GND**.
5. Wire the LED **Anode** to **GPIO 23**, **Cathode** to **GND**.
6. Paste the code into the editor and select **Interpreted Mode**.
7. Click **Run**.
8. In the HC-05 widget serial input, type **L** and press Enter — LED toggles.
9. Type **B** — buzzer fires a short beep.
10. Type **S** — servo sweeps 0°→180°→0°.
11. Type **O** — all outputs reset.

## Expected Output

Serial Monitor:
```
Voice Command Controller Ready.
L=LED  B=Buzzer  S=Servo  O=All off
CMD: LED ON
CMD: BUZZER beep
CMD: SERVO sweep start
Servo sweep complete.
CMD: ALL OFF
```

## Expected Canvas Behavior

* Typing **L** toggles the Red LED indicator on the ARIES canvas component.
* Typing **B** lights the Buzzer widget for ~500 ms then extinguishes it.
* Typing **S** animates the Servo widget needle from 0° to 180° and back.
* Typing **O** resets all widget states simultaneously.

## Code Walkthrough

| Line | Check / Action |
| --- | --- |
| `Serial.begin(9600)` | Opens UART at 9600 baud — HC-05 factory default rate. |
| `Serial.available() > 0` | Non-blocking check; reads only when a Bluetooth byte has arrived. |
| `char cmd = Serial.read()` | Consumes the first available byte as a character. |
| `cmd == 'L'` | Dispatch block; both upper and lower case accepted for resilience. |
| `buzzTimer++` | Counts loop ticks; 25 ticks × 20 ms = 500 ms beep duration. |
| `sweepStep >= 3` | Advances servo angle every 60 ms (3 ticks × 20 ms) for smooth motion. |
| `servoDir = -1` | Reverses sweep when 180° reached; stops and clears flag at 0°. |
| `analogWrite(PIN_SERVO, pwmVal)` | Maps 0–180° to 0–255 PWM range used by MbedO servo widget. |

## Hardware & Safety Concept

**Bluetooth Serial UART Bridge:** The HC-05 acts as a transparent UART-to-Bluetooth bridge. A paired smartphone sends a character; it appears as a byte on HC-05 TXD connected to ARIES RX (GPIO 0). The ARIES reads it with `Serial.available()` / `Serial.read()` identically to a wired USB connection. Speech recognition apps convert spoken words (e.g., "LED on") to text and map the first character to a command. This event-driven architecture never blocks `loop()`, allowing simultaneous buzzer timing and servo motion via independent state machines.

**Voltage Safety:** HC-05 logic is 3.3 V. Its TXD output is 3.3 V, compatible with ARIES 3.3 V GPIO inputs. ARIES TX output is also 3.3 V, safe for HC-05 RXD. Always verify your specific HC-05 module datasheet; some breakout boards include internal level shifters.

## Try This! (Challenges)

1. **Add a Green LED channel**: Implement command `'G'` to toggle **LED_G** (GPIO 24) independently, giving two-colour voice-controlled lighting.
2. **Beep-pattern command feedback**: After each command, configure the buzzer to emit distinct short beep counts (1 = LED, 2 = servo) using the buzzTimer state machine to distinguish actions audibly without adding extra functions.

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| No characters in Serial Monitor | HC-05 TX/RX wires swapped | Confirm HC-05 TXD → GPIO 0 and HC-05 RXD → GPIO 1. |
| Buzzer stays on permanently | Loop tick count never clears | Verify `delay(20)` is present at loop bottom so `buzzTimer` advances. |
| Servo jumps to maximum instantly | PWM mapping uses integer division | Ensure `servoAngle / 180.0` uses float literal `180.0` not integer `180`. |
| Speech app commands not recognized | App sends whole words, not chars | Configure app to send single-letter shortcuts per phrase in its settings. |

## Mode Notes

This project runs in MbedO **interpreted C++ mode**.

## Related Projects

- [185 - Bluetooth LED Controller](185-bluetooth-led-controller.md)
- [187 - Smart Irrigation System](187-smart-irrigation-system.md)
- [195 - Full Home Automation Hub](195-full-home-automation-hub.md)
