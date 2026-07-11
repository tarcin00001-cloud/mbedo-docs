# 135 - ESP32 Robotic Arm Position Memory Log

Build a robotic arm teaching station that records manual joint angles to an array buffer when a record button is pressed, and replays the saved sequence when a play button is clicked.

## Goal
Learn how to store multi-dimensional coordinates in arrays, construct play/record sequence states, and disable manual controls during macro playbacks.

## What You Will Build
Three potentiometers control three servos (Base on GPIO 13, Shoulder on GPIO 12, Gripper on GPIO 14). A Record button is connected to GPIO 4, and a Play button to GPIO 15. Pressing Record saves the current angles to an array (up to 10 slots). Pressing Play disables the pots and sweeps the servos through the recorded sequence.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometers (3) | `potentiometer` | Yes | Yes |
| Servos (3) | `servo` | Yes | Yes |
| Tactile Push Buttons (2) | `button` | Yes | Yes |
| 10 kΩ Resistors (2) | `resistor` | No | Yes |
| External Power Supply (5V, 2A) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Pot 1 / 2 / 3 | Wipers | GPIO34 / 35 / 32 | Yellow | Base/Shl/Claw analog inputs |
| Servo 1 / 2 / 3 | Signals | GPIO13 / 12 / 14 | Orange/Blue/Green | Joint PWM control |
| Button A (Record) | Pin 1 / Pin 2 | 3V3 / GPIO4 | Red / Yellow | Save position frame |
| Resistor A (10k) | Leg 1 / Leg 2 | GPIO4 / GND | White / Black | Pull-down for Record button |
| Button B (Play) | Pin 1 / Pin 2 | 3V3 / GPIO15 | Red / Green | Playback sequence |
| Resistor B (10k) | Leg 1 / Leg 2 | GPIO15 / GND | White / Black | Pull-down for Play button |

> **Wiring tip:** Connect both buttons with 10 kΩ pull-down resistors to keep inputs stable. Use external power for the three servos.

## Code
```cpp
// Robotic Arm Position Memory Log (Play/Record)
#include <ESP32Servo.h>

const int POT_BASE = 34;
const int POT_SHL = 35;
const int POT_CLAW = 32;

const int SERVO_BASE = 13;
const int SERVO_SHL = 12;
const int SERVO_CLAW = 14;

const int BTN_RECORD = 4;
const int BTN_PLAY = 15;

Servo baseServo, shoulderServo, clawServo;

// Memory Array
const int MAX_FRAMES = 10;
int savedPositions[MAX_FRAMES][3]; // Columns: [Base, Shoulder, Claw]
int frameCount = 0;

bool lastRecState = LOW;
bool lastPlayState = LOW;

void setup() {
  Serial.begin(115200);
  
  pinMode(BTN_RECORD, INPUT);
  pinMode(BTN_PLAY, INPUT);
  
  baseServo.attach(SERVO_BASE);
  shoulderServo.attach(SERVO_SHL);
  clawServo.attach(SERVO_CLAW);
  
  baseServo.write(90);
  shoulderServo.write(90);
  clawServo.write(10);
  
  Serial.println("Teaching station online. Set position and press RECORD.");
}

void loop() {
  bool recPressed = (digitalRead(BTN_RECORD) == HIGH);
  bool playPressed = (digitalRead(BTN_PLAY) == HIGH);
  
  // 1. Record Position Frame
  if (recPressed && !lastRecState) {
    delay(50); // Debounce
    if (frameCount < MAX_FRAMES) {
      int baseVal = map(analogRead(POT_BASE), 0, 4095, 0, 180);
      int shlVal  = map(analogRead(POT_SHL), 0, 4095, 0, 180);
      int clawVal = map(analogRead(POT_CLAW), 0, 4095, 10, 90);
      
      savedPositions[frameCount][0] = baseVal;
      savedPositions[frameCount][1] = shlVal;
      savedPositions[frameCount][2] = clawVal;
      
      Serial.print(">> RECORDED FRAME "); Serial.print(frameCount);
      Serial.print(": [B="); Serial.print(baseVal);
      Serial.print(", S="); Serial.print(shlVal);
      Serial.print(", C="); Serial.print(clawVal);
      Serial.println("]");
      
      frameCount++;
    } else {
      Serial.println("Memory Full! Cannot record more steps.");
    }
  }
  lastRecState = recPressed;
  
  // 2. Playback Sequence
  if (playPressed && !lastPlayState) {
    delay(50); // Debounce
    if (frameCount > 0) {
      Serial.println("!!! PLAYBACK SEQUENCE STARTING !!!");
      
      for (int i = 0; i < frameCount; i++) {
        int targetBase = savedPositions[i][0];
        int targetShl  = savedPositions[i][1];
        int targetClaw = savedPositions[i][2];
        
        Serial.print("Executing Step "); Serial.print(i);
        Serial.print(": [B="); Serial.print(targetBase);
        Serial.print(", S="); Serial.print(targetShl);
        Serial.print(", C="); Serial.print(targetClaw);
        Serial.println("]");
        
        // Command servos to target positions
        baseServo.write(targetBase);
        shoulderServo.write(targetShl);
        clawServo.write(targetClaw);
        
        delay(1200); // Wait for mechanical travel to complete
      }
      Serial.println("Playback complete. Manual control re-enabled.");
    } else {
      Serial.println("No recorded frames to play!");
    }
  }
  lastPlayState = playPressed;
  
  // 3. Live Manual Control (only active when not pressing buttons)
  if (!recPressed && !playPressed) {
    int liveBase = map(analogRead(POT_BASE), 0, 4095, 0, 180);
    int liveShl  = map(analogRead(POT_SHL), 0, 4095, 0, 180);
    int liveClaw = map(analogRead(POT_CLAW), 0, 4095, 10, 90);
    
    baseServo.write(liveBase);
    shoulderServo.write(liveShl);
    clawServo.write(liveClaw);
    
    delay(20); // 50Hz live tracking
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, three **Potentiometers**, three **Servos**, and two **Buttons** onto the canvas.
2. Wire the parts as listed in the tables.
3. Paste the code and click **Run**.
4. Adjust the potentiometers to a target pose, click the Record button (GPIO 4) to save a frame. Move dials to another pose, click Record.
5. Click the Play button (GPIO 15) to watch the recorded sequence play automatically.

## Expected Output
Serial Monitor:
```
Teaching station online. Set position and press RECORD.
>> RECORDED FRAME 0: [B=45, S=120, C=10]
>> RECORDED FRAME 1: [B=135, S=90, C=90]
!!! PLAYBACK SEQUENCE STARTING !!!
Executing Step 0: [B=45, S=120, C=10]
Executing Step 1: [B=135, S=90, C=90]
Playback complete. Manual control re-enabled.
```

## Expected Canvas Behavior
* Moving the potentiometer sliders rotates the corresponding servos live.
* Clicking the Record button widget saves the current angles to the next slot.
* Clicking the Play button widget disables the potentiometer inputs and commands the servos to cycle through the saved angles, pausing 1.2 seconds at each step.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `savedPositions[10][3]` | Double array buffer storing up to 10 frames of 3 joint angles. |
| `baseServo.write(targetBase)` | Commands the servo to move to the stored coordinate during playback. |
| `!recPressed && !playPressed` | Isolates manual control so that adjustments are disabled during macro playbacks. |

## Hardware & Safety Concept: Teaching Modes and Robotic Macro Recorders
Industrial robotic arms utilize a **Teach Pendant**. Operators steer the arm manually to targets and press buttons to record coordinate waypoints (joints or tool tip positions). Once the path is logged, the arm switches to automated playback mode, moving at high speeds with high precision, repeating the macro sequence continuously.

## Try This! (Challenges)
1. **Loop Playback Mode**: Modify the play button logic so that holding the button down plays the sequence in a continuous loop until released.
2. **Clear Memory Button**: Add a routine where pressing both buttons together resets `frameCount = 0` to clear the log.
3. **Transition Speed Ramping**: Smooth the transition between frames by sweeping the angles slowly instead of stepping instantly.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Record button saves multiple steps on one click | Missing edge-trigger check | Ensure the code evaluates `recPressed && !lastRecState` to capture only the press event |
| Playback moves servos too fast/violently | Delay between steps too short | Increase the step delay to 1500 ms to give the arm mechanical time to travel |
| Memory limits exceeded | Array overflow | Check `frameCount < MAX_FRAMES` before saving to prevent memory corruption |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [134 - ESP32 3-Axis Robotic Arm Claw Controller](134-esp32-3-axis-robotic-arm-claw-controller.md)
- [133 - ESP32 2-Axis Robotic Arm Joint Controller](133-esp32-2-axis-robotic-arm-joint-controller.md)
- [136 - ESP32 SD Card Data Logger](136-esp32-sd-card-data-logger.md)
