# 172 - ESP32 RTOS Mutex Shared Resource Protector

Build an RTOS application on the ESP32 that runs two independent tasks (reading a DHT22 sensor and reading a potentiometer) that write to a shared 16x2 I2C LCD screen, using a FreeRTOS Mutex Semaphore to prevent write collisions.

## Goal
Learn how to create and use FreeRTOS Mutex Semaphores to protect shared hardware resources (like I2C or SPI buses) from write collisions in a multi-threaded environment.

## What You Will Build
A DHT22 sensor is connected to GPIO 4. A potentiometer is on GPIO 34. A 16x2 I2C LCD is on GPIO 21/22. Two tasks run in parallel: Task 1 updates temperature on LCD Row 0 every 2 seconds; Task 2 updates potentiometer readings on LCD Row 1 every 500 ms. A **Mutex (Mutual Exclusion)** semaphore ensures only one task can write to the LCD at a time, avoiding data corruption.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Temp sensor input |
| Potentiometer | Wiper | GPIO34 | White | Potentiometer input |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (Shared) |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The I2C lines are shared by the LCD. Power the LCD from 5V Vin for stable backlight brightness.

## Code
```cpp
// RTOS Mutex Shared Resource Protector (Shared I2C LCD)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

const int DHT_PIN = 4;
const int POT_PIN = 34;

#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Mutex Semaphore Handle
SemaphoreHandle_t lcdMutex;

// Task 1: Read DHT22 and write to LCD Row 0
void TaskDHT(void *pvParameters) {
  dht.begin();
  
  while (1) {
    float temp = dht.readTemperature();
    float hum = dht.readHumidity();
    
    if (!isnan(temp) && !isnan(hum)) {
      // Try to acquire the LCD Mutex. Wait indefinitely if busy
      if (xSemaphoreTake(lcdMutex, portMAX_DELAY) == pdTRUE) {
        // Critical Section: Writing to the shared LCD
        lcd.setCursor(0, 0);
        lcd.print("T:"); lcd.print(temp, 1);
        lcd.print("C H:"); lcd.print(hum, 0);
        lcd.print("%     "); // Clear trailing space
        
        Serial.println("[DHT Task] Acquired Mutex and wrote to Row 0.");
        
        // Release the Mutex immediately after write
        xSemaphoreGive(lcdMutex); 
      }
    }
    
    // Read every 2 seconds
    vTaskDelay(pdMS_TO_TICKS(2000)); 
  }
}

// Task 2: Read Potentiometer and write to LCD Row 1
void TaskPotentiometer(void *pvParameters) {
  while (1) {
    int rawVal = analogRead(POT_PIN);
    int percent = map(rawVal, 0, 4095, 0, 100);
    
    // Try to acquire the LCD Mutex. Wait indefinitely if busy
    if (xSemaphoreTake(lcdMutex, portMAX_DELAY) == pdTRUE) {
      // Critical Section: Writing to the shared LCD
      lcd.setCursor(0, 1);
      lcd.print("Pot: "); lcd.print(percent);
      lcd.print("% (Raw:"); lcd.print(rawVal);
      lcd.print(")   "); 
      
      Serial.println("[POT Task] Acquired Mutex and wrote to Row 1.");
      
      // Release the Mutex
      xSemaphoreGive(lcdMutex); 
    }
    
    // Read every 500 ms
    vTaskDelay(pdMS_TO_TICKS(500)); 
  }
}

void setup() {
  Serial.begin(115200);
  
  // Initialize LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Mutex Protector");
  lcd.setCursor(0, 1);
  lcd.print("Initializing...");
  delay(1500);
  lcd.clear();
  
  // Create Mutex Semaphore
  lcdMutex = xSemaphoreCreateMutex();
  
  if (lcdMutex != NULL) {
    Serial.println("LCD Mutex created successfully.");
    
    // Create Tasks
    xTaskCreate(
      TaskDHT,
      "DHT Task",
      2048,
      NULL,
      1, // Low priority
      NULL
    );
    
    xTaskCreate(
      TaskPotentiometer,
      "POT Task",
      2048,
      NULL,
      1, // Low priority
      NULL
    );
  } else {
    Serial.println("Error creating LCD Mutex!");
    while(1) {} // Halt if mutex creation failed
  }
}

void loop() {
  // Empty loop
  vTaskDelay(pdMS_TO_TICKS(1000));
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, **Potentiometer**, and **16x2 I2C LCD** onto the canvas.
2. Wire DHT22 to **GPIO4**, Potentiometer to **GPIO34**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the potentiometer and change the DHT22 temperature values. Watch both lines of the LCD update smoothly without flickering or corrupting.

## Expected Output
Serial Monitor:
```
LCD Mutex created successfully.
[POT Task] Acquired Mutex and wrote to Row 1.
[POT Task] Acquired Mutex and wrote to Row 1.
[DHT Task] Acquired Mutex and wrote to Row 0.
```

LCD Display:
```
T:24.5C H:45%
Pot: 32% (Raw:1310)
```

## Expected Canvas Behavior
* The LCD displays the live temperature on Row 0 and the live potentiometer value on Row 1.
* The LCD updates cleanly and does not flicker or display garbled characters.
* If you slide both widgets at the same time, the display remains stable.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `xSemaphoreCreateMutex()` | Creates a binary mutex semaphore to protect a shared resource. |
| `xSemaphoreTake(lcdMutex, ...)` | Blocks the task until the mutex is free. Once free, locks the mutex to gain exclusive access. |
| `xSemaphoreGive(lcdMutex)` | Releases the mutex, allowing other tasks to acquire it. |

## Hardware & Safety Concept: Resource Contention and Mutex Locks
In a multi-tasking operating system, multiple tasks share hardware resources (such as I2C buses, SPI ports, or memory buffers). If Task A interrupts Task B while Task B is midway through sending an I2C packet to the LCD, the data will collide, corrupting the display or locking the I2C bus. A **Mutex (Mutual Exclusion)** act as a lock: before writing to the shared resource, a task must lock the mutex, and unlock it when finished, preventing other tasks from accessing the resource.

## Try This! (Challenges)
1. **Mutex Timeout warning**: Instead of waiting indefinitely (`portMAX_DELAY`), set a timeout of 100 ms. If a task fails to acquire the mutex, print a warning to the Serial Monitor.
2. **Serial printing Mutex**: Protect the `Serial.print` commands using a second mutex to prevent mixed print characters.
3. **Emergency LCD override**: Create a high-priority task (Priority 4) that bypasses the standard loop to write an alert immediately if a button is pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD display freezes or displays garbled characters | Mutex not released | Ensure every call to `xSemaphoreTake` is matched by a call to `xSemaphoreGive` in all execution paths |
| A task hangs and never runs | Deadlock | Avoid locking multiple mutexes inside the same task in different orders (causes deadlocks) |
| LCD does not initialize | Incorrect address | Verify that the I2C address matches `0x27` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [171 - ESP32 Multi-tasking RTOS Scheduler Demo](171-esp32-multi-tasking-rtos-scheduler-demo.md)
- [173 - ESP32 RTOS Queue Data Producer-Consumer](173-esp32-rtos-queue-data-producer-consumer.md) (Next project)
- [58 - ESP32 16×2 I2C LCD Print Text](../intermediate/58-esp32-16x2-i2c-lcd-print-text.md)
