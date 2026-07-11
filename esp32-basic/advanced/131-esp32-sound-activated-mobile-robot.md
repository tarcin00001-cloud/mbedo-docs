# 131 - ESP32 Sound Activated Mobile Robot

Build a voice/clap-activated mobile robot that toggles its drive state (Forward / Stopped) each time it detects a loud noise threshold via a microphone sound sensor.

## Goal
Learn how to capture rapid sound envelope triggers, implement software debounce states to ignore echo noises, and toggle DC motor status.

## What You Will Build
An L298N driver controls two DC motors (Left: 18/19/5, Right: 21/22/23). An electret microphone sound module DO (Digital Output) is connected to GPIO 4. The robot starts stopped. Detecting a loud clap toggles the robot state to drive Forward. The next clap stops it.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motors (2) | `dc_motor` | Yes | Yes |
| Microphone Sound Sensor Module | `sound_sensor` | Yes | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 / IN2 / ENA | GPIO18 / GPIO19 / GPIO5 | Yellow/Green/Orange | Left Motor |
| L298N Module | IN3 / IN4 / ENB | GPIO21 / GPIO22 / GPIO23 | Blue/Purple/White | Right Motor |
| Sound Sensor | OUT (Digital) | GPIO4 | Yellow | Audio envelope digital input |
| Sound Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| L298N Module | GND | GND | Black | Common Ground |

> **Wiring tip:** Standard sound sensors output active-LOW digital triggers when sound exceeds the onboard potentiometer comparator threshold (LOW = sound detected). Connect OUT to GPIO 4.

## Code
```cpp
// Sound Activated Mobile Robot
const int IN1 = 18;
const int IN2 = 19;
const int ENA = 5;

const int IN3 = 21;
const int IN4 = 22;
const int ENB = 23;

const int SOUND_PIN = 4;

bool isMoving = false;
bool lastSoundState = HIGH;
unsigned long lastTriggerTime = 0;
const int DEBOUNCE_DELAY_MS = 500; // Ignore echos within 500ms

void robotForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
  Serial.println("Robot State: FORWARD");
}

void robotStop() {
  digitalWrite(ENA, LOW);
  digitalWrite(ENB, LOW);
  Serial.println("Robot State: STOPPED");
}

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT); pinMode(ENA, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT); pinMode(ENB, OUTPUT);
  
  pinMode(SOUND_PIN, INPUT_PULLUP);
  
  robotStop(); // Start stopped
  Serial.println("Sound Activated Robot Online. Clap to toggle.");
}

void loop() {
  // Read sound sensor digital out (LOW = Sound Detected)
  bool soundTriggered = (digitalRead(SOUND_PIN) == LOW);
  unsigned long now = millis();
  
  // Detect falling edge (HIGH to LOW transition)
  if (soundTriggered && lastSoundState == HIGH) {
    // Apply debounce time window to ignore clashing motor echo noise
    if (now - lastTriggerTime > DEBOUNCE_DELAY_MS) {
      isMoving = !isMoving;
      
      if (isMoving) {
        robotForward();
      } else {
        robotStop();
      }
      
      lastTriggerTime = now;
    }
  }
  
  lastSoundState = soundTriggered ? LOW : HIGH;
  delay(10); // Fast scan rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, two **DC Motors**, and **Sound Sensor** onto the canvas.
2. Wire Sound OUT to **GPIO4**, and motor pins to **18, 19, 5, 21, 22, 23**.
3. Paste the code and click **Run**.
4. Click the Sound Sensor widget to simulate a loud sound. Watch the wheels start spinning.
5. Click the widget again. Watch the wheels stop.

## Expected Output
Serial Monitor:
```
Sound Activated Robot Online. Ready to scan.
Robot State: FORWARD
Robot State: STOPPED
```

## Expected Canvas Behavior
* At boot, both motor widgets are stopped.
* Clicking the virtual sound sensor widget once toggles both motor widgets to spin forward (clockwise).
* Clicking the sound widget again halts both motors.
* Clicking triggers must be separated by at least 500 ms.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `digitalRead(SOUND_PIN) == LOW` | Matches the active-LOW output of the comparator when sound amplitude exceeds the threshold. |
| `now - lastTriggerTime > 500` | Filters out motor noise and echoes so the state doesn't toggle multiple times per clap. |
| `isMoving = !isMoving` | Toggles the movement state boolean. |

## Hardware & Safety Concept: Acoustic Feedback Debouncing
DC motors generate physical vibrations and high-frequency electrical brush noise that can trigger microphone sensors. When building sound-activated mobile systems, you must use **debouncing locks** in code: immediately after a sound triggers a state change, lock the sensor inputs for 500 ms to allow the motors to start spinning and the sound reflections to decay before listening for the next command.

## Try This! (Challenges)
1. **Status indicator LED**: Add an LED on GPIO 12 that lights up when the robot is in the FORWARD state.
2. **Double Clap Turn**: Modify the code to drive forward on one clap, and pivot turn on a double-clap (two claps within 1 second).
3. **Sound Peak Logger**: Connect the analog output (AO) of the microphone to GPIO 34 to log environmental noise peaks.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot starts moving and immediately stops | Sensor threshold too sensitive | Adjust the blue trimmer potentiometer on the sound module |
| Motors don't respond | Sensor digital output pin float | Ensure `INPUT_PULLUP` is declared to hold input pins high |
| Robot triggers from its own motor startup | Missing debounce lock | Verify that the `DEBOUNCE_DELAY_MS` variable is at least 500 ms |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [42 - ESP32 Sound Sensor Clap Detector](../beginner/42-esp32-sound-sensor-clap-detector.md)
- [121 - ESP32 Dual Motor Forward/Reverse Drive](121-esp32-dual-motor-forward-reverse-drive.md)
- [122 - ESP32 Mobile Robot Left/Right Steering](122-esp32-mobile-robot-left-right-steering.md)
