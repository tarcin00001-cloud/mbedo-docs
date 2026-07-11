# 24 - Button counter (output logs to Serial)

Track the number of times a push button is pressed and print the running count to the Serial Monitor of the VEGA ARIES v3 board.

## Goal
Learn how to implement accumulation variables, perform rising or falling edge state checks, and output formatted diagnostic logging to the Serial Monitor.

## What You Will Build
A push button is wired to `GPIO 16` with the internal pull-up resistor enabled. The program monitors the button's logic level. When it transition from HIGH to LOW (falling edge), the code increments an integer counter variable and prints the updated count to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Terminal 1 | GPIO 16 | Blue | Digital input signal pin |
| Push Button | Terminal 2 | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Since the button is wired to the internal pull-up pin on the VEGA ARIES v3 board, no external resistors are required. Simply connect one terminal of the button to GPIO 16 and the other to GND.

## Code
```cpp
// Button Counter - VEGA ARIES v3
const int BUTTON_PIN = 16;

int lastButtonState = HIGH; // Tracks previous button state
int pressCount = 0;         // Accumulator variable for button presses

void setup() {
  // Initialize Serial communication at 115200 baud
  Serial.begin(115200);
  
  // Configure the button pin with internal pull-up
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  
  // Print initial startup message
  Serial.println("System Initialized.");
  Serial.println("Press the button to increment the counter.");
}

void loop() {
  int currentButtonState = digitalRead(BUTTON_PIN);
  
  // Detect falling edge (transition from HIGH to LOW)
  if (lastButtonState == HIGH && currentButtonState == LOW) {
    pressCount = pressCount + 1; // Increment counter
    
    // Log the event and current count to the Serial Monitor
    Serial.print("Button Pressed! Count: ");
    Serial.println(pressCount);
    
    delay(150); // Debounce delay to prevent double-counting
  }
  
  lastButtonState = currentButtonState; // Save state for next iteration
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and a **Push Button** onto the canvas.
2. Wire the Push Button between **GPIO 16** and **GND**.
3. Paste the code into the editor.
4. Click **Run**.
5. Open the **Serial Monitor** tab in MbedO.
6. Click the Push Button on the canvas. Observe the output log in the Serial Monitor increments by 1 with each press.

## Expected Output
Serial Monitor:
```
System Initialized.
Press the button to increment the counter.
Button Pressed! Count: 1
Button Pressed! Count: 2
Button Pressed! Count: 3
Button Pressed! Count: 4
```

## Expected Canvas Behavior
* Pressing the push button does not trigger any visible LED states on the canvas.
* The button widget changes to its pressed state, and text updates appear in the Serial Monitor interface.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Serial.begin(115200)` | Starts Serial data transmission at a baud rate of 115200 bits per second. |
| `pressCount = pressCount + 1` | Increments the button press tally variable by 1. |
| `Serial.print(...)` | Sends the label text to the Serial output buffer without adding a newline. |
| `Serial.println(...)` | Sends the count value followed by a carriage return and newline. |

## Hardware & Safety Concept
* **Serial Communication (UART)**: Universal Asynchronous Receiver-Transmitter (UART) is the hardware block responsible for serial communication. The THEJAS32 SoC converts internal digital states into serial data packets and sends them over the USB interface to your computer.
* **Baud Rate Mismatch**: Baud rate dictates the clock speed of serial data transmission. If the board transmits at 115200 baud but the Serial Monitor is configured to 9600 baud, the output will appear as scrambled, unreadable characters (garbage text). Always ensure both are matched.

## Try This! (Challenges)
1. **Counter Threshold Alert**: Add the onboard Red LED (`LED_R` on GPIO 23) and make it light up if the button count reaches 5 or more presses.
2. **Reverse Counter**: Add a second push button on GPIO 17. Configure it so that pressing Button 1 increments the count, and pressing Button 2 decrements the count.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Serial Monitor is empty | Program not running | Ensure you clicked the "Run" button and that the program has compiled successfully |
| Scrambled characters in Serial Monitor | Baud rate mismatch | Make sure your terminal software or simulator setting matches the `115200` baud rate declared in code |
| Count increases by more than 1 per press | Debounce delay too short | Increase the debounce delay in the loop from `150` to `200` or `250` milliseconds |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [23 - Reed switch magnetic sensor](23-reed-switch-magnetic-sensor.md)
- [25 - Double click button filter](25-double-click-button-filter.md) (Next project)
