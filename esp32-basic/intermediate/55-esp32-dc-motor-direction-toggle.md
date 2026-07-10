# 55 - ESP32 DC Motor Direction Toggle

Use a single push button to toggle a DC motor between forward, stop, and reverse states in sequence using the L298N motor driver.

## Goal
Learn how to implement a multi-state state machine with a button, cycling a DC motor through three states (FORWARD → STOP → REVERSE → FORWARD) on each button press.

## What You Will Build
An L298N motor driver with a DC motor. A push button on GPIO 4 cycles the motor through three states. GPIO 18/19 set direction; GPIO 5 (ENA) enables the channel.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motor (3–12 V) | `dc_motor` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Pin 1 (supply side) | 3V3 | Red | Supply voltage |
| Push Button | Pin 2 (signal side) | GPIO4 | Yellow | State-cycle input |
| 10 kΩ Resistor | Leg 1 | GPIO4 | White | Pull-down resistor |
| 10 kΩ Resistor | Leg 2 | GND | Black | Ground reference |
| L298N Module | IN1 | GPIO18 | Yellow | Direction bit 1 |
| L298N Module | IN2 | GPIO19 | Green | Direction bit 2 |
| L298N Module | ENA | GPIO5 | Orange | Enable — HIGH = running |
| L298N Module | GND | GND | Black | Shared ground with ESP32 |
| L298N Module | 12V (VCC) | External 6–12 V + | Red | Motor power supply |
| L298N Module | GND (power) | External supply − | Black | Motor supply ground |
| DC Motor | Terminal A | OUT1 | Blue | Motor lead 1 |
| DC Motor | Terminal B | OUT2 | White | Motor lead 2 |

> **Wiring tip:** The 10 kΩ pull-down on GPIO 4 ensures a clean LOW reading when the button is open. The motor state machine changes on every rising edge, so one press = one state advance regardless of how long the button is held.

## Code
```cpp
// DC Motor Direction Toggle — 3-state machine via button
const int BTN_PIN = 4;
const int IN1     = 18;
const int IN2     = 19;
const int ENA     = 5;

// States: 0 = STOP, 1 = FORWARD, 2 = REVERSE
int motorState = 0;
bool lastBtn   = false;

const char* stateNames[] = {"STOP", "FORWARD", "REVERSE"};

void applyState(int state) {
  switch (state) {
    case 0:  // STOP
      digitalWrite(ENA, LOW);
      break;
    case 1:  // FORWARD
      digitalWrite(IN1, HIGH);
      digitalWrite(IN2, LOW);
      digitalWrite(ENA, HIGH);
      break;
    case 2:  // REVERSE
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, HIGH);
      digitalWrite(ENA, HIGH);
      break;
  }
  Serial.print("Motor state: "); Serial.println(stateNames[state]);
}

void setup() {
  pinMode(BTN_PIN, INPUT);
  pinMode(IN1,     OUTPUT);
  pinMode(IN2,     OUTPUT);
  pinMode(ENA,     OUTPUT);
  applyState(0);   // Start in STOP

  Serial.begin(115200);
  Serial.println("Motor Direction Toggle ready — press button to cycle states.");
}

void loop() {
  bool btn = digitalRead(BTN_PIN);

  if (btn && !lastBtn) {
    // Rising edge — advance to next state
    motorState = (motorState + 1) % 3;   // Cycles 0→1→2→0
    applyState(motorState);
  }

  lastBtn = btn;
  delay(50);   // Debounce
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Button**, **L298N**, and **DC Motor** onto the canvas.
2. Connect Button → **GPIO4**, IN1 → **GPIO18**, IN2 → **GPIO19**, ENA → **GPIO5**.
3. Paste the code and click **Run**.
4. Click the button widget to cycle: FORWARD → STOP → REVERSE → FORWARD.

## Expected Output
Serial Monitor:
```
Motor Direction Toggle ready — press button to cycle states.
Motor state: FORWARD
Motor state: STOP
Motor state: REVERSE
Motor state: FORWARD
```

## Expected Canvas Behavior
* Motor starts stopped.
* First button click: motor spins forward.
* Second click: motor stops.
* Third click: motor spins in reverse.
* Fourth click: back to forward. Cycle repeats.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `int motorState = 0` | Tracks current state: 0=STOP, 1=FORWARD, 2=REVERSE. |
| `motorState = (motorState + 1) % 3` | Modulo arithmetic cycles 0→1→2→0→1→2 indefinitely. |
| `switch (state)` | Applies the correct IN1/IN2/ENA combination for the requested state. |
| `if (btn && !lastBtn)` | Rising-edge detection — triggers once per button press regardless of hold time. |
| `delay(50)` | 50 ms debounce gap. |

## Hardware & Safety Concept: State Machines in Embedded Control
A **state machine** is a programming pattern where a system can be in one of a finite set of states, and transitions between states are triggered by events. In this project: the current state is `motorState`, the event is a button rising edge, and the transition is `(state + 1) % 3`. State machines are the standard architecture for: traffic light controllers, washing machine cycles, industrial PLC programs, communication protocols (UART, SPI), and robot behaviour trees. They are preferred over deeply nested `if/else` chains because: each state's behaviour is clearly isolated in a `switch` case, transitions are explicit, and adding new states requires only a new `case` block.

## Try This! (Challenges)
1. **LED indicators**: Add three LEDs (GPIO 12, 13, 14) — one for each state — and light the appropriate LED.
2. **Auto-return to STOP**: If the motor is in FORWARD or REVERSE and no button is pressed for 10 seconds, automatically return to STOP.
3. **Long-press shortcut**: Detect a long press (> 1 second) and jump directly to STOP from any state.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor skips states rapidly | Button bounce | Increase debounce delay to 100 ms |
| Motor never moves | ENA not driven HIGH in FORWARD/REVERSE cases | Verify `digitalWrite(ENA, HIGH)` is in both running cases |
| Only forward works | IN2 not wired to GPIO19 | Re-seat GPIO19 wire in IN2 terminal |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [53 - ESP32 DC Motor Start/Stop](53-esp32-dc-motor-start-stop.md)
- [54 - ESP32 DC Motor Speed Scaling](54-esp32-dc-motor-speed-scaling.md)
- [22 - ESP32 Touch Toggle Controller](../beginner/22-esp32-touch-toggle-controller.md)
