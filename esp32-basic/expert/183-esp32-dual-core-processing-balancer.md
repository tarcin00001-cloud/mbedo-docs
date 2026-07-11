# 183 - ESP32 Dual-core Processing Balancer

Build a dual-core load balancer on the ESP32 that runs a high-frequency mathematical Kalman filter loop pinned to Core 0, and a graphical OLED HUD update loop pinned to Core 1, demonstrating symmetric multiprocessing (SMP).

## Goal
Learn how to pin tasks to specific ESP32 CPU cores (`xTaskCreatePinnedToCore`), share data thread-safely across cores, and balance processing loads.

## What You Will Build
An MPU-6050 sensor and an SSD1306 OLED share the I2C bus (GPIO 21/22). A buzzer is on GPIO 13. Task 1 is pinned to **Core 0**, running a 100Hz loop that reads the IMU and runs a Kalman filter. Task 2 is pinned to **Core 1**, running a 25Hz loop that updates the OLED with a graphical tilt bar and controls the alarm buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MPU-6050 6-DOF IMU Sensor | `mpu6050` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (Shared) |
| SSD1306 OLED | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (Shared) |
| Active Buzzer | VCC (+) | GPIO13 | Blue | Alarm output |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Verify that the buzzer pin is connected directly to GPIO 13.

## Code
```cpp
// Dual-core Processing Balancer (Multi-Core task execution)
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
Adafruit_MPU6050 mpu;

const int BUZZER_PIN = 13;

// Shared volatile variables accessed across CPU cores
volatile float sharedFilteredAngle = 0.0;
SemaphoreHandle_t angleMutex; // Mutex to protect shared float operations

// Simple 1D Kalman Filter state variables
float Q_angle = 0.001;
float Q_bias  = 0.003;
float R_measure = 0.03;
float kAngle = 0.0;
float kBias = 0.0;
float kP[2][2] = { {0,0}, {0,0} };

float runKalmanFilter(float newAngle, float newRate, float dt) {
  float rate = newRate - kBias;
  kAngle += dt * rate;
  kP[0][0] += dt * (dt * kP[1][1] - kP[0][1] - kP[1][0] + Q_angle);
  kP[0][1] -= dt * kP[1][1];
  kP[1][0] -= dt * kP[1][1];
  kP[1][1] += Q_bias * dt;

  float S = kP[0][0] + R_measure;
  float K[2];
  K[0] = kP[0][0] / S;
  K[1] = kP[1][0] / S;

  float y = newAngle - kAngle;
  kAngle += K[0] * y;
  kBias  += K[1] * y;

  float P00_temp = kP[0][0];
  float P01_temp = kP[0][1];
  kP[0][0] -= K[0] * P00_temp;
  kP[0][1] -= K[0] * P01_temp;
  kP[1][0] -= K[1] * P00_temp;
  kP[1][1] -= K[1] * P01_temp;

  return kAngle;
}

// ==========================================
// CORE 0 TASK: High-Speed Sensor Filtering
// ==========================================
void TaskCore0_Sensor(void *pvParameters) {
  Serial.print("Sensor Task running on Core "); Serial.println(xPortGetCoreID());
  
  unsigned long lastTime = micros();
  
  while (1) {
    unsigned long now = micros();
    float dt = (now - lastTime) / 1000000.0;
    if (dt <= 0.0) dt = 0.01;
    lastTime = now;
    
    // Read raw IMU values
    sensors_event_t a, g, temp;
    mpu.getEvent(&a, &g, &temp);
    
    float rawAngle = atan2(a.acceleration.y, a.acceleration.z) * 180.0 / M_PI;
    float gyroRate = g.gyro.x * 180.0 / M_PI;
    
    // Run Kalman filter
    float filtered = runKalmanFilter(rawAngle, gyroRate, dt);
    
    // Write value to shared variable using Mutex protection
    if (xSemaphoreTake(angleMutex, portMAX_DELAY) == pdTRUE) {
      sharedFilteredAngle = filtered;
      xSemaphoreGive(angleMutex);
    }
    
    // Run at ~100Hz (10 ms delay)
    vTaskDelay(pdMS_TO_TICKS(10)); 
  }
}

// ==========================================
// CORE 1 TASK: OLED HUD Rendering & Buzzer Alarm
// ==========================================
void TaskCore1_OLED(void *pvParameters) {
  Serial.print("OLED Task running on Core "); Serial.println(xPortGetCoreID());
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  while (1) {
    float angleToDraw = 0.0;
    
    // Read shared angle using Mutex
    if (xSemaphoreTake(angleMutex, portMAX_DELAY) == pdTRUE) {
      angleToDraw = sharedFilteredAngle;
      xSemaphoreGive(angleMutex);
    }
    
    // Draw graphical HUD
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.print("DUAL-CORE BALANCER");
    display.setCursor(0, 10);
    display.print("Core 1 Core 0 Shared");
    
    // Draw horizontal tilt bar (center: 64, 40)
    int centerX = 64;
    int centerY = 40;
    int barLength = map((int)angleToDraw, -45, 45, -40, 40);
    barLength = constrain(barLength, -40, 40);
    
    display.drawRect(24, 36, 80, 8, SSD1306_WHITE);
    display.fillRect(centerX, 38, barLength, 4, SSD1306_WHITE);
    
    display.setCursor(0, 52);
    display.print("Angle: "); display.print(angleToDraw, 1); display.print(" deg");
    display.display();
    
    // Safety Alarm: Sound buzzer if tilt > 30 degrees
    if (abs(angleToDraw) > 30.0) {
      digitalWrite(BUZZER_PIN, HIGH);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
    }
    
    // Run at ~25Hz (40 ms delay) to save I2C bandwidth
    vTaskDelay(pdMS_TO_TICKS(40)); 
  }
}

void setup() {
  Serial.begin(115200);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED allocation failed!");
    while(1) {}
  }
  
  if (!mpu.begin()) {
    Serial.println("MPU-6050 init failed!");
    while(1) {}
  }
  
  // Create Mutex
  angleMutex = xSemaphoreCreateMutex();
  
  if (angleMutex == NULL) {
    Serial.println("Mutex creation failed!");
    while(1) {}
  }
  
  // Create Pinned Tasks:
  // Pinned to Core 0 (Low-level fast tasks)
  xTaskCreatePinnedToCore(
    TaskCore0_Sensor,
    "Sensor Task",
    4096,
    NULL,
    2, // Higher priority
    NULL,
    0  // Pin to Core 0
  );
  
  // Pinned to Core 1 (UI and Application tasks)
  xTaskCreatePinnedToCore(
    TaskCore1_OLED,
    "OLED Task",
    4096,
    NULL,
    1, // Low priority
    NULL,
    1  // Pin to Core 1
  );
  
  Serial.println("Dual core initialization complete.");
}

void loop() {
  // Empty loop
  vTaskDelay(pdMS_TO_TICKS(1000));
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MPU-6050**, **SSD1306 OLED**, and **Buzzer** onto the canvas.
2. Wire shared I2C bus to **GPIO21/GPIO22** and Buzzer to **GPIO13**.
3. Paste the code and click **Run**.
4. Slide the MPU-6050 Y acceleration slider. Watch the horizontal tilt bar on the OLED follow the tilt.
5. Inspect the Serial Monitor. Verify that the two tasks log running on separate CPU cores (Core 0 and Core 1).

## Expected Output
Serial Monitor:
```
Dual core initialization complete.
Sensor Task running on Core 0
OLED Task running on Core 1
```

OLED Display Layout:
```
DUAL-CORE BALANCER
Core 1 Core 0 Shared
      ┌──────┬──────┐
      │      █████  │ (Tilt bar indicator moving right)
      └──────┴──────┘
Angle: 18.5 deg
```

## Expected Canvas Behavior
* The OLED display draws a horizontal frame box with a centered tilt bar.
* Adjusting the simulated IMU slider moves the bar right or left.
* If the slider angle exceeds 30 degrees, the buzzer widget pulses green.
* The system runs smoothly without lag.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `xTaskCreatePinnedToCore(...)` | Registers the task and pins its execution exclusively to the specified core (0 or 1). |
| `xPortGetCoreID()` | Returns the index of the CPU core (0 or 1) executing the current task. |
| `xSemaphoreTake(angleMutex, ...)` | Locks access to the shared float variable while reading/writing. |

## Hardware & Safety Concept: Symmetric Multiprocessing (SMP) on Dual-Core CPUs
Most microcontrollers contain a single CPU core. The ESP32 contains **two independent cores** sharing the same memory space. If a single core handles both high-frequency sensor filtering (Kalman filter) and slow graphics rendering, the screen updates can block sensor processing. Using **Dual-Core Processing**, Core 0 handles the mathematical calculations, while Core 1 handles the user interface. This ensures critical sensors are updated on time.

## Try This! (Challenges)
1. **Dynamic Core Swapper**: Add a button on GPIO 4 that dynamically swaps the tasks between Core 0 and Core 1.
2. **Multi-core benchmark**: Toggle two pins at high speed on separate cores and measure the speed difference using an oscilloscope.
3. **Core overload warning**: Log task execution times to warn if Core 0 utilization exceeds 90%.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| System crashes immediately on boot | Memory allocation error | Increase stack size allocations in `xTaskCreatePinnedToCore` to 4096 bytes |
| Screen updates are sluggish | Bus conflicts | Limit OLED updates to 25Hz to prevent locking the shared I2C bus |
| Variable values are corrupted | Shared variable accessed without mutex protection | Ensure all read/write operations on the shared float variable are protected by the mutex |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [171 - ESP32 Multi-tasking RTOS Scheduler Demo](171-esp32-multi-tasking-rtos-scheduler-demo.md)
- [172 - ESP32 RTOS Mutex Shared Resource Protector](172-esp32-rtos-mutex-shared-resource-protector.md)
- [154 - ESP32 OLED Kalman Filter Demo](154-esp32-oled-kalman-filter-demo.md)
