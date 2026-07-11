# 191 - ESP32 Autonomous Robot HUD Node

Build an autonomous obstacle-avoiding mobile robot on the ESP32 that pins high-priority motor driving and ultrasonic distance checking to Core 0, and pins background OLED status HUD updates to Core 1, using mutexes to protect shared states.

## Goal
Learn how to design multi-core architectures for mobile robotics, run real-time sensor processing and motor outputs on Core 0, and update user interfaces on Core 1.

## What You Will Build
An L298N driver controls two DC motors (IN1: 18, IN2: 19, ENA: 5, IN3: 25, IN4: 26, ENB: 27). An HC-SR04 sonar sensor is on GPIO 12/13, and an SSD1306 OLED is on I2C. Core 0 runs the steering and obstacle avoidance loop. Core 1 updates the OLED with a graphical dashboard showing the distance and active driving state.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motors (2) | `dc_motor` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic_sensor` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |
| External Power Supply (6-9V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 / IN2 / ENA | GPIO18 / GPIO19 / GPIO5 | Yellow/Green/Orange | Left motor speed controls |
| L298N Module | IN3 / IN4 / ENB | GPIO25 / GPIO26 / GPIO27 | Yellow/Green/Orange | Right motor speed controls |
| HC-SR04 Sonar | Trig / Echo | GPIO12 / GPIO13 | Orange / Yellow | Ultrasonic distance lines |
| SSD1306 OLED | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Power both DC motors and the L298N driver from an external battery pack to prevent voltage drops from resetting the ESP32.

## Code
```cpp
// Autonomous Robot HUD Node (Core 0 Motors + Core 1 OLED)
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// Motor Pins
const int IN1 = 18;
const int IN2 = 19;
const int ENA = 5;
const int IN3 = 25;
const int IN4 = 26;
const int ENB = 27;

// Sonar Pins
const int TRIG_PIN = 12;
const int ECHO_PIN = 13;

// Shared variables (protected by Mutex)
volatile float sharedDistance = 100.0;
volatile const char* sharedState = "START";
SemaphoreHandle_t robotMutex;

// Motor movement routines
void setLeftMotor(int speed, bool forward) {
  digitalWrite(IN1, forward ? HIGH : LOW);
  digitalWrite(IN2, forward ? LOW : HIGH);
  ledcWrite(0, speed);
}

void setRightMotor(int speed, bool forward) {
  digitalWrite(IN3, forward ? HIGH : LOW);
  digitalWrite(IN4, forward ? LOW : HIGH);
  ledcWrite(1, speed);
}

void stopRobot() {
  ledcWrite(0, 0);
  ledcWrite(1, 0);
}

// ==========================================
// CORE 0 TASK: High-Priority Navigation loop
// ==========================================
void TaskCore0_Nav(void *pvParameters) {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  
  // Setup PWM channels
  ledcSetup(0, 2000, 8);
  ledcAttachPin(ENA, 0);
  ledcSetup(1, 2000, 8);
  ledcAttachPin(ENB, 1);
  
  stopRobot();
  vTaskDelay(pdMS_TO_TICKS(1000));
  
  while (1) {
    // 1. Measure Distance
    digitalWrite(TRIG_PIN, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG_PIN, LOW);
    
    long duration = pulseIn(ECHO_PIN, HIGH, 30000); // 30ms timeout (~5m)
    float distance = (duration > 0) ? (duration * 0.0343 / 2.0) : 400.0;
    
    const char* activeState = "FORWARD";
    
    // 2. Obstacle Avoidance Logic
    if (distance < 20.0) {
      // Obstacle detected: stop and pivot
      activeState = "PIVOT";
      stopRobot();
      vTaskDelay(pdMS_TO_TICKS(200));
      
      // Pivot left
      setLeftMotor(150, false);
      setRightMotor(150, true);
      vTaskDelay(pdMS_TO_TICKS(600));
      stopRobot();
      vTaskDelay(pdMS_TO_TICKS(100));
    } else {
      // Safe: drive forward
      activeState = "FORWARD";
      setLeftMotor(160, true);
      setRightMotor(160, true);
    }
    
    // 3. Write telemetry to shared variables
    if (xSemaphoreTake(robotMutex, portMAX_DELAY) == pdTRUE) {
      sharedDistance = distance;
      sharedState = activeState;
      xSemaphoreGive(robotMutex);
    }
    
    vTaskDelay(pdMS_TO_TICKS(60)); // ~16Hz check rate
  }
}

// ==========================================
// CORE 1 TASK: Background HUD display
// ==========================================
void TaskCore1_HUD(void *pvParameters) {
  while (1) {
    float distToDraw = 0.0;
    const char* stateToDraw = "UNKNOWN";
    
    // Read shared variables
    if (xSemaphoreTake(robotMutex, portMAX_DELAY) == pdTRUE) {
      distToDraw = sharedDistance;
      stateToDraw = sharedState;
      xSemaphoreGive(robotMutex);
    }
    
    // Draw HUD Dashboard
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.print("ROBOT DUAL-CORE HUD");
    display.drawFastHLine(0, 10, 128, SSD1306_WHITE);
    
    display.setCursor(0, 15);
    display.print("State: ");
    display.setTextSize(2);
    display.print(stateToDraw);
    
    display.setTextSize(1);
    display.setCursor(0, 35);
    display.print("Obstacle Dist: ");
    display.print(distToDraw, 1);
    display.print(" cm");
    
    // Draw a visual warning bar if distance is low
    int barWidth = map((int)distToDraw, 0, 100, 0, 120);
    barWidth = constrain(barWidth, 0, 120);
    display.drawRect(4, 52, 120, 8, SSD1306_WHITE);
    display.fillRect(6, 54, barWidth, 4, SSD1306_WHITE);
    
    display.display();
    
    vTaskDelay(pdMS_TO_TICKS(100)); // 10Hz screen update rate
  }
}

void setup() {
  Serial.begin(115200);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED allocation failed!");
    while(1) {}
  }
  
  robotMutex = xSemaphoreCreateMutex();
  
  if (robotMutex == NULL) {
    Serial.println("Mutex creation failed!");
    while(1) {}
  }
  
  // Create Pinned Tasks:
  // Navigation on Core 0 (Low-level priority task)
  xTaskCreatePinnedToCore(
    TaskCore0_Nav,
    "Nav Task",
    4096,
    NULL,
    2, // Higher priority
    NULL,
    0  // Pin to Core 0
  );
  
  // HUD update on Core 1 (Application/UI task)
  xTaskCreatePinnedToCore(
    TaskCore1_HUD,
    "HUD Task",
    4096,
    NULL,
    1, // Lower priority
    NULL,
    1  // Pin to Core 1
  );
}

void loop() {
  // Empty loop
  vTaskDelay(pdMS_TO_TICKS(1000));
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, two **Motors**, **HC-SR04**, and **SSD1306 OLED** onto the canvas.
2. Wire motors to L298N, sonar to **GPIO12/GPIO13**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the sonar distance widget below 20 cm. Watch the robot stop and pivot (reversing motor directions) while the OLED state updates.
5. Set the distance slider to 50 cm. Watch the robot drive forward.

## Expected Output
Serial Monitor:
* Real-time dual-core execution logs.

OLED Display Layout:
```
ROBOT DUAL-CORE HUD
───────────────────
State: FORWARD
Obstacle Dist: 52.4 cm
 ┌────────────────┐
 │████████        │ (Distance bar graph)
 └────────────────┘
```

## Expected Canvas Behavior
* At startup, the OLED initializes.
* If the sonar distance is > 20 cm, both motor widgets spin clockwise (forward). The OLED shows `State: FORWARD`.
* Dragging the sonar distance below 20 cm stops both motors, turns one motor counter-clockwise and the other clockwise (pivoting), and updates the OLED to `State: PIVOT`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `xTaskCreatePinnedToCore(...)` | Pins navigation controls to Core 0 and user interface updates to Core 1. |
| `xSemaphoreTake(robotMutex, ...)` | Locks access to shared variables (distance and state) to prevent race conditions. |
| `pulseIn(...)` | Measures the duration of the ultrasonic echo pulse to calculate distance. |

## Hardware & Safety Concept: Multi-Core task allocation in Robotics
Mobile robots operating autonomously must respond quickly to sensor events (like detecting a wall or cliff). If the microcontroller is busy drawing complex graphics on an OLED or writing logs to an SD card, it can miss sensor triggers, causing a crash. Using **Dual-Core Processing**, navigation tasks are pinned to Core 0 (running at high priority), while user interface updates run on Core 1. This prevents screen rendering delays from lagging motor control.

## Try This! (Challenges)
1. **Emergency Bumper override**: Add a limit switch bumper on GPIO 4 that triggers a hardware interrupt, stopping all motors instantly.
2. **Speedometer integration**: Connect a rotary encoder (Project 187) and display the robot's speed in RPM on the OLED HUD.
3. **Bluetooth remote steering**: Add a Bluetooth module (Project 168) allowing remote control override via serial commands.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The robot does not turn or moves sluggishly | Low motor voltage | Power the L298N motor driver from an external battery pack |
| OLED screen freezes | Mutex deadlock | Ensure all calls to `xSemaphoreTake` are matched by a call to `xSemaphoreGive` |
| Sonar distance reads static 0 or 400 | Sonar pins miswired | Verify Trig is connected to GPIO 12 and Echo to GPIO 13 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [124 - ESP32 Obstacle Avoidance Robot](../intermediate/124-esp32-obstacle-avoidance-robot.md)
- [183 - ESP32 Dual-core Processing Balancer](183-esp32-dual-core-processing-balancer.md)
- [172 - ESP32 RTOS Mutex Shared Resource Protector](172-esp32-rtos-mutex-shared-resource-protector.md)
