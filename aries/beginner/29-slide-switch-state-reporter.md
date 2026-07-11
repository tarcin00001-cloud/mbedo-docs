# 29 - Slide Switch state reporter

Monitor the physical state of a slide switch and report state changes to the Serial Monitor of the VEGA ARIES v3 board.

## Goal
Learn how to interface slide switches, read digital inputs, implement state-change detection (detecting both rising and falling edges), and print messages to the Serial Monitor.

## What You Will Build
A slide switch is connected to `GPIO 17` and configured with the internal pull-up resistor. When the switch is slid to the ON position, it connects the pin to GND (LOW). When slid to the OFF position, it breaks the connection, causing the internal pull-up to pull the voltage to 3.3V (HIGH). The ARIES board detects when the switch position changes and prints "Switch State: CLOSED (ON)" or "Switch State: OPEN (OFF)" to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Slide Switch | `slide_switch` (SPDT) | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Slide Switch | Pin 1 (Left) | GND | Black | Closed circuit connection |
| Slide Switch | Pin 2 (Middle) | GPIO 17 | Green | Digital sense connection |
| Slide Switch | Pin 3 (Right) | NC | None | Left unconnected |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** A slide switch is typically an SPDT (Single Pole Double Throw) switch. Connect the center pin (Pin 2) to GPIO 17. Connect one of the outer pins (Pin 1) to GND. Leave Pin 3 unconnected. When the slider is moved toward Pin 1, the circuit closes to GND.

## Code
```cpp
// Slide Switch State Reporter - VEGA ARIES v3
const int SWITCH_PIN = 17;

int lastSwitchState = -1; // Initialize to -1 to force an initial print in setup/loop

void setup() {
  // Initialize Serial communication at 115200 baud
  Serial.begin(115200);
  
  // Configure slide switch pin as input with internal pull-up
  pinMode(SWITCH_PIN, INPUT_PULLUP);
  
  // Initial print message
  Serial.println("System Initialized.");
  Serial.println("Monitoring slide switch state...");
}

void loop() {
  int currentSwitchState = digitalRead(SWITCH_PIN);
  
  // Check if the switch state has changed
  if (currentSwitchState != lastSwitchState) {
    if (currentSwitchState == LOW) {
      Serial.println("Switch State: CLOSED (ON)");
    } else {
      Serial.println("Switch State: OPEN (OFF)");
    }
    
    // Save the new state
    lastSwitchState = currentSwitchState;
  }
  
  delay(100); // Polling delay to prevent terminal spamming
  }
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and a **Slide Switch** onto the canvas.
2. Wire Pin 2 (Middle) of the Slide Switch to **GPIO 17**.
3. Wire Pin 1 (Left) of the Slide Switch to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Open the **Serial Monitor** tab.
7. Click the Slide Switch widget on the canvas to slide it back and forth.
8. Observe the state changes printed in the Serial Monitor.

## Expected Output
Serial Monitor:
```
System Initialized.
Monitoring slide switch state...
Switch State: OPEN (OFF)
Switch State: CLOSED (ON)
Switch State: OPEN (OFF)
```

## Expected Canvas Behavior
* Sliding the switch on the canvas does not trigger any onboard LEDs.
* Position changes are instantly reported as text logs in the Serial Monitor tab.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(SWITCH_PIN, INPUT_PULLUP)` | Activates internal pull-up on GPIO 17, keeping the pin HIGH when the switch is slid to the open side. |
| `currentSwitchState != lastSwitchState` | Compares the current switch position to the previous position, detecting any state change. |
| `currentSwitchState == LOW` | Checks if the slider is in the closed position, connecting GPIO 17 to GND. |

## Hardware & Safety Concept
* **SPDT Slide Switch Operation**: Single Pole Double Throw (SPDT) switches have a single common terminal (Pin 2) that toggles between two other terminals (Pin 1 and Pin 3). By leaving Pin 3 unconnected, we treat the switch as a simple SPST (Single Pole Single Throw) ON/OFF switch.
* **State Change Detection**: Printing to the Serial Monitor on every loop iteration would spam the serial buffer and slow down the program. Checking `currentSwitchState != lastSwitchState` ensures that we only transmit data when the user physically changes the switch position.

## Try This! (Challenges)
1. **Safety Shutdown Sync**: Connect the onboard Red LED (`LED_R` on GPIO 23). Program the LED to turn ON when the switch is OPEN (OFF) to represent a system lock state, and turn OFF when CLOSED (ON).
2. **Double Switch Interlock**: Add a push button on GPIO 16. Program the code so that pressing the push button only toggles an LED if the slide switch is in the CLOSED (ON) position.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Serial Monitor spams ON/OFF messages continuously | Pull-up resistor not configured | Ensure you have declared `INPUT_PULLUP` instead of `INPUT` in `setup()` |
| No change in Serial Monitor when sliding | Wired outer pins incorrectly | Ensure the middle pin of the slide switch is connected to GPIO 17, and one outer pin to GND |
| No initial status printed | Serial baud mismatch | Match the terminal speed to the code's 115200 baud rate |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [28 - Laser receiver sensor trigger](28-laser-receiver-sensor-trigger.md)
- [30 - Sound sensor clapper (Clap → LED toggle)](30-sound-sensor-clapper.md) (Next project)
