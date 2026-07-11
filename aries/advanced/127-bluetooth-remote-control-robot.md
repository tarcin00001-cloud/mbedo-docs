# 127 - Bluetooth Remote Control Robot

Control a mobile robot wirelessly using a smartphone or laptop terminal by streaming commands through an HC-05 Bluetooth module to the ARIES board.

## Goal
Learn how to interface wireless transceiver modules (HC-05), process serial data packets asynchronously on `Serial2`, and translate character commands into motor actions.

## What You Will Build
A wireless remote-controlled robot. The ARIES board listens on its secondary hardware serial port (`Serial2`) for commands sent from a paired Bluetooth device (such as a smartphone app or Bluetooth terminal). The robot decodes single-character commands: `'F'` (Forward), `'B'` (Backward), `'L'` (Left), `'R'` (Right), and `'S'` (Stop), and moves accordingly, returning status messages back to the user.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05` | Yes | Yes |
| L298N Motor Driver Module | `l298n` | Yes | Yes |
| 2x DC Toy Motors | `dc_motor` | Yes | Yes |
| External 6V-12V Power Source | `power_supply` | Optional | Yes (battery pack) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 Module | VCC | 5V | Red | Bluetooth module operating power |
| HC-05 Module | GND | GND | Black | Ground reference |
| HC-05 Module | TXD | RX2 (GP21) | Blue | HC-05 TX to ARIES RX2 |
| HC-05 Module | RXD | TX2 (GP20) | Yellow | HC-05 RX to ARIES TX2 |
| L298N Module | VCC / GND | 5V / GND | Red / Black | Motor driver power |
| L298N Module | IN1 / IN2 | GPIO 14 / 15 | Orange / Brown | Left motor control |
| L298N Module | IN3 / IN4 | GPIO 13 / 12 | Green / Blue | Right motor control |

> **Wiring tip:** Cross the serial lines correctly. The ARIES board transmits data on **TX2 (GP20)**, which connects to the HC-05 **RXD** pin. The ARIES board receives data on **RX2 (GP21)**, which connects to the HC-05 **TXD** pin.

## Code
```cpp
// Bluetooth Remote Control Robot - VEGA ARIES v3
const int IN1 = 14; // Left Motor Forward
const int IN2 = 15; // Left Motor Backward
const int IN3 = 13; // Right Motor Forward
const int IN4 = 12; // Right Motor Backward

void setup() {
  // Configure motor control pins as outputs
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  // Stop motors initially
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);

  // Initialize primary USB Serial for local monitoring
  Serial.begin(9600);
  
  // Initialize secondary Hardware Serial2 for Bluetooth module
  Serial2.begin(9600);

  Serial.println("Bluetooth Control Robot online. Waiting for commands...");
  Serial2.println("Robot Online. Send F, B, L, R, or S.");
}

void loop() {
  // Check if data is available from the Bluetooth module
  if (Serial2.available() > 0) {
    // Read the incoming character command
    char cmd = Serial2.read();
    
    // Mirror to USB monitor
    Serial.print("Received Command: ");
    Serial.println(cmd);

    // Process commands
    if (cmd == 'F' || cmd == 'f') {
      // Move Forward
      Serial2.println("Action: Forward");
      digitalWrite(IN1, HIGH);
      digitalWrite(IN2, LOW);
      digitalWrite(IN3, HIGH);
      digitalWrite(IN4, LOW);
    } 
    else if (cmd == 'B' || cmd == 'b') {
      // Move Backward
      Serial2.println("Action: Backward");
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, HIGH);
      digitalWrite(IN3, LOW);
      digitalWrite(IN4, HIGH);
    } 
    else if (cmd == 'L' || cmd == 'l') {
      // Spin Turn Left
      Serial2.println("Action: Turn Left");
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, HIGH);
      digitalWrite(IN3, HIGH);
      digitalWrite(IN4, LOW);
    } 
    else if (cmd == 'R' || cmd == 'r') {
      // Spin Turn Right
      Serial2.println("Action: Turn Right");
      digitalWrite(IN1, HIGH);
      digitalWrite(IN2, LOW);
      digitalWrite(IN3, LOW);
      digitalWrite(IN4, HIGH);
    } 
    else if (cmd == 'S' || cmd == 's') {
      // Stop
      Serial2.println("Action: Stop");
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, LOW);
      digitalWrite(IN3, LOW);
      digitalWrite(IN4, LOW);
    }
    else {
      // Unknown command
      Serial2.print("Unknown command: ");
      Serial2.println(cmd);
    }
  }
  
  delay(50); // Small delay to prevent CPU spinning
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **HC-05 Bluetooth Module**, **L298N Motor Driver**, and two **DC Motors** onto the canvas.
2. Wire the HC-05 module: **VCC** to **5V**, **GND** to **GND**, **TXD** to **RX2 (GP21)**, and **RXD** to **TX2 (GP20)**.
3. Wire the L298N module: **VCC** to **5V**, **GND** to **GND**, and input pins **IN1-IN4** to **GPIO 14, 15, 13, 12**. Connect output pins to the DC motors.
4. Paste the C++ code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run**. Open the simulated Bluetooth terminal and send character commands (`F`, `B`, `L`, `R`, `S`) to drive the robot.

## Expected Output
Serial Monitor (Local Debug):
```
Bluetooth Control Robot online. Waiting for commands...
Received Command: F
Received Command: R
Received Command: S
```
Bluetooth Terminal (Remote Application):
```
Robot Online. Send F, B, L, R, or S.
Action: Forward
Action: Turn Right
Action: Stop
```

## Expected Canvas Behavior
* Sending `'F'` makes the motors spin forward.
* Sending `'B'` makes both motors spin backward.
* Sending `'L'` or `'R'` causes the wheels to rotate in opposite directions to execute a spin turn.
* Sending `'S'` halts both motors.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Serial2.begin(9600)` | Registers the secondary hardware UART controller on ARIES at a 9600 baud rate. |
| `Serial2.available() > 0` | Evaluates if any bytes have been received in the UART hardware buffer. |
| `char cmd = Serial2.read()` | Extracts a single byte from the hardware receiver FIFO queue. |
| `cmd == 'F' \|\| cmd == 'f'` | Evaluates if the received character is a forward command. |

## Hardware & Safety Concept
* **Bluetooth UART Protocol**: The HC-05 module communicates with the ARIES board using the asynchronous serial UART standard. This protocol uses a constant baud rate (9600 bps), 8 data bits, no parity bit, and 1 stop bit (8N1). It behaves just like the default USB serial line but redirects data over wireless RF (2.4 GHz Bluetooth) waves.
* **Logic Level Tolerances**: Many HC-05 modules require 5V for power (VCC), but their logic pins (TXD/RXD) operate at 3.3V. The ARIES board works on 3.3V logic, meaning it can safely receive from the HC-05 TXD pin and output 3.3V to the HC-05 RXD pin without requiring external voltage dividers or level translation.

## Try This! (Challenges)
1. **LED Remote Toggle**: Add code to toggle the ARIES onboard Red LED (`LED_R`) ON and OFF when command `'X'` and `'Y'` are received over Bluetooth, respectively.
2. **Speed Step Control**: Introduce commands `'1'`, `'2'`, and `'3'` to set the robot's speed using PWM via ENA/ENB (similar to Project 123) to scale the speeds to 100, 180, and 255.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The Bluetooth app fails to connect or send data | HC-05 RX/TX pins crossed or incorrect baud rate | Verify that ARIES TX2 connects to HC-05 RXD and ARIES RX2 connects to HC-05 TXD. Ensure `Serial2.begin(9600)` is used. |
| Received command prints as random symbols | Ground loop or speed mismatch | Make sure the HC-05 shares a common ground (GND) wire with the ARIES board. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [120 - Bluetooth DHT22 Telemetry Broadcast](../intermediate/120-bluetooth-dht22-telemetry-broadcast.md)
- [128 - Bluetooth Robot with Crash Override](128-bluetooth-robot-with-crash-override.md)
