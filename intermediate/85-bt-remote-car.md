# 85 - BT Remote Car

Control a two-wheeled robot chassis wirelessly by sending direction commands over Bluetooth.

## Goal
Learn how to decode custom serial characters ('F', 'B', 'L', 'R', 'S') sent via Bluetooth to steer and drive an H-Bridge motor vehicle.

## What You Will Build
The Arduino polls the SoftwareSerial Bluetooth link. When it receives a command character, it coordinates the L298N pins:
- **'F' (Forward)**: Both wheels spin forward.
- **'B' (Backward)**: Both wheels spin backward.
- **'L' (Left Turn)**: Right wheel spins forward, Left wheel stops or reverses.
- **'R' (Right Turn)**: Left wheel spins forward, Right wheel stops or reverses.
- **'S' (Stop)**: Both wheels stop.

**Why pins D4 to D10?** Pins D2/D3 run the Bluetooth port. Pins D9/D10 are PWM channels to control speeds. Pins D4, D5, D6, and D7 are direction control lines (IN1, IN2, IN3, IN4).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05_bluetooth` | Yes | Yes |
| L298N Driver | `l298n_driver` | Yes | Yes |
| DC Motor | `dc_motor` | Yes (2 required) | Yes (2 required) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-05 Module | TXD | D2 | Software RX |
| HC-05 Module | RXD | D3 | Software TX (Use divider on hardware) |
| HC-05 Module | VCC | 5V | Power supply |
| HC-05 Module | GND | GND | Ground reference |
| L298N Driver | ENA | D9 | Left Motor Speed (PWM) |
| L298N Driver | IN1 | D4 | Left Motor Direction 1 |
| L298N Driver | IN2 | D5 | Left Motor Direction 2 |
| L298N Driver | IN3 | D6 | Right Motor Direction 1 |
| L298N Driver | IN4 | D7 | Right Motor Direction 2 |
| L298N Driver | ENB | D10 | Right Motor Speed (PWM) |
| L298N Driver | VIN | 5V | Power supply |
| L298N Driver | GND | GND | Ground reference |
| Left Motor | + / - | OUT1 / OUT2 | Driven by Left Channel |
| Right Motor | + / - | OUT3 / OUT4 | Driven by Right Channel |

## Code
```cpp
#include <SoftwareSerial.h>

SoftwareSerial BT(2, 3); // RX = D2, TX = D3

// Left Motor Control Pins
const int ENA = 9; 
const int IN1 = 4;
const int IN2 = 5;

// Right Motor Control Pins
const int IN3 = 6;
const int IN4 = 7;
const int ENB = 10;

void setup() {
  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(ENB, OUTPUT);
  
  // Stop motors at startup
  stopRobot();
  
  Serial.begin(9600);
  BT.begin(9600);
  Serial.println("Bluetooth Remote Car Controller Ready");
}

void loop() {
  if (BT.available() > 0) {
    char command = BT.read();
    
    Serial.print("Command: ");
    Serial.println(command);
    
    // Process movement commands
    switch (command) {
      case 'F':
      case 'f':
        moveForward();
        BT.println("Driving Forward");
        break;
        
      case 'B':
      case 'b':
        moveBackward();
        BT.println("Driving Backward");
        break;
        
      case 'L':
      case 'l':
        turnLeft();
        BT.println("Turning Left");
        break;
        
      case 'R':
      case 'r':
        turnRight();
        BT.println("Turning Right");
        break;
        
      case 'S':
      case 's':
        stopRobot();
        BT.println("Stopped");
        break;
        
      default:
        // Ignore unrecognized characters
        break;
    }
  }
}

// ---- Movement Helper Functions ----

void moveForward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  analogWrite(ENA, 200);
  
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  analogWrite(ENB, 200);
}

void moveBackward() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  analogWrite(ENA, 200);
  
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  analogWrite(ENB, 200);
}

void turnLeft() {
  // Spin right wheel forward, reverse left wheel
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  analogWrite(ENA, 150);
  
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  analogWrite(ENB, 150);
}

void turnRight() {
  // Spin left wheel forward, reverse right wheel
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  analogWrite(ENA, 150);
  
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  analogWrite(ENB, 150);
}

void stopRobot() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  analogWrite(ENA, 0);
  
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  analogWrite(ENB, 0);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-05 Bluetooth Module**, **L298N Driver**, and **two DC Motors** onto the canvas.
2. Wire HC-05: **TXD** to **D2**, **RXD** to **D3**, **VCC** to **5V**, and **GND** to **GND**.
3. Wire L298N inputs: **ENA** to **D9**, **IN1** to **D4**, **IN2** to **D5**, **IN3** to **D6**, **IN4** to **D7**, **ENB** to **D10**, **VIN** to **5V**, and **GND** to **GND**.
4. Wire L298N outputs: **OUT1/OUT2** to the Left Motor, and **OUT3/OUT4** to the Right Motor.
5. Paste the code into the editor.
6. Select the **Compiled Arduino Uno** runtime.
7. Click **Build and Run** (or **Run**).
8. Open the **Terminal** tab in the bottom dock.
9. Send characters `'F'`, `'B'`, `'L'`, `'R'`, and `'S'` over the simulated Bluetooth port, and watch the wheels change spin directions.

## Expected Output

Terminal:
```
Bluetooth Remote Car Controller Ready
Command: F
Command: L
Command: S
...
```
The motors rotate dynamically based on the command characters swiped.

## Expected Canvas Behavior

| Sent Command | Left Motor | Right Motor | Vehicle action |
| --- | --- | --- | --- |
| 'F' | Clockwise (Forward) | Clockwise (Forward) | Drives straight ahead |
| 'B' | Counter-clockwise (Reverse) | Counter-clockwise (Reverse) | Drives straight back |
| 'L' | Counter-clockwise (Reverse) | Clockwise (Forward) | Spins left |
| 'R' | Clockwise (Forward) | Counter-clockwise (Reverse) | Spins right |
| 'S' | Stopped | Stopped | Stops |

The motors respond instantly upon command reception.

## Code Walkthrough

| Function / Helper | What It Does |
| --- | --- |
| `turnLeft()` | Inverts the polarity of the left motor relative to the right motor, causing the vehicle to perform a pivot spin turn. |
| `stopRobot()` | Disables both H-bridge enable channels, bringing both motors to a quick, passive stop. |

## Hardware & Safety Concept: Differential Steering
Differential steering is the method of steering a wheeled vehicle by varying the relative rate of rotation of its wheels.
- To drive straight, both wheels spin forward at the same speed.
- To spin left, the right wheel spins forward while the left wheel spins backward.
This is the steering system used by tracked military tanks, skid-steer loaders, and most two-wheeled educational robots because it has a zero turning radius and does not require complex steering racks.

## Try This! (Challenges)
1. **Soft Stops**: Modify the `stopRobot()` function to turn the direction pins LOW and let the speed decelerate slowly (coasting stop).
2. **Speed Controls**: Add commands `'1'`, `'2'`, and `'3'` to set the robot drive speed to Low (100 PWM), Medium (180 PWM), and Max (255 PWM).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot turns right when commanded left | Left and right motor wires swapped | Swap the output wire bundles at the L298N driver terminals OUT1/OUT2 and OUT3/OUT4. |
| Code does not respond to Bluetooth | Incorrect runtime selected | Verify that you have selected the **Compiled Arduino Uno** runtime. |

## Mode Notes
This project **requires the Compiled Arduino Uno** runtime because custom byte parsing and nested case switch actions are not processed by the interpreted mode runner.

## Related Projects
- [72 - BT Serial Bridge](72-bt-serial-bridge.md)
- [73 - BT LED Control](73-bt-led-control.md)
- [84 - Robot Forward Back](84-robot-forward-back.md)
