# 192 - ESP32 Multi-Sensor Environmental HUD

Build a multi-tasking environmental monitoring station on the ESP32 that runs independent FreeRTOS tasks to query a DHT22 sensor and a BMP180 pressure sensor, synchronizes readings, and uses a page-switching task to display them on an LCD.

## Goal
Learn how to schedule multiple sensor query tasks with different period rates, protect shared variables and the I2C bus using mutexes, and build an interactive UI task.

## What You Will Build
A DHT22 is on GPIO 4. A BMP180 sensor and a 16x2 LCD share the I2C bus (GPIO 21/22). A push button is on GPIO 12. Task 1 reads the DHT22 every 2 seconds. Task 2 reads the BMP180 every 3 seconds. Task 3 monitors the button and updates the LCD, displaying DHT22 stats on Page 0 and BMP180 stats on Page 1. A Mutex protects the shared I2C bus.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| BMP180 Pressure Sensor | `bmp180` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Temp & Humidity input |
| BMP180 Sensor | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (Shared) |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (Shared) |
| Push Button | Pin 1 / Pin 2 | GPIO12 / 3V3 | Orange / Red | Page toggle (active-HIGH) |
| Push Button | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Connect the push button with an external 10 kΩ pull-down resistor to GPIO 12.

## Code
```cpp
// Multi-Sensor Environmental HUD (DHT22 + BMP180 + LCD FreeRTOS)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>
#include <Adafruit_BMP085.h>

const int DHT_PIN = 4;
const int BUTTON_PIN = 12;

#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);
Adafruit_BMP085 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Shared telemetry variables
volatile float dhtTemp = 0.0;
volatile float dhtHum = 0.0;
volatile float bmpTemp = 0.0;
volatile float bmpPress = 0.0; // in hPa

// Shared state variable
volatile int lcdPage = 0;

// Mutex to protect shared variables and I2C bus accesses
SemaphoreHandle_t i2cMutex;

// Task 1: Read DHT22 every 2 seconds
void TaskReadDHT(void *pvParameters) {
  dht.begin();
  
  while (1) {
    float temp = dht.readTemperature();
    float hum = dht.readHumidity();
    
    if (!isnan(temp) && !isnan(hum)) {
      // Lock shared variables write access
      if (xSemaphoreTake(i2cMutex, portMAX_DELAY) == pdTRUE) {
        dhtTemp = temp;
        dhtHum = hum;
        xSemaphoreGive(i2cMutex);
        Serial.print("[DHT Task] Updated readings: "); Serial.print(temp, 1); Serial.println("C");
      }
    }
    
    vTaskDelay(pdMS_TO_TICKS(2000)); // Sample every 2 seconds
  }
}

// Task 2: Read BMP180 every 3 seconds
void TaskReadBMP(void *pvParameters) {
  // Lock I2C during initialization
  if (xSemaphoreTake(i2cMutex, portMAX_DELAY) == pdTRUE) {
    if (!bmp.begin()) {
      Serial.println("[BMP Task] BMP180 Init Failed!");
    }
    xSemaphoreGive(i2cMutex);
  }
  
  while (1) {
    float temp = 0.0;
    float press = 0.0;
    
    // Acquire mutex before reading from the shared I2C bus
    if (xSemaphoreTake(i2cMutex, portMAX_DELAY) == pdTRUE) {
      temp = bmp.readTemperature();
      press = bmp.readPressure() / 100.0; // Convert Pa to hPa
      
      bmpTemp = temp;
      bmpPress = press;
      xSemaphoreGive(i2cMutex);
      Serial.print("[BMP Task] Updated readings: "); Serial.print(press, 1); Serial.println("hPa");
    }
    
    vTaskDelay(pdMS_TO_TICKS(3000)); // Sample every 3 seconds
  }
}

// Task 3: Monitor button and update LCD
void TaskDisplay(void *pvParameters) {
  pinMode(BUTTON_PIN, INPUT);
  
  bool lastBtnState = LOW;
  
  while (1) {
    // Read page change button
    bool btnState = (digitalRead(BUTTON_PIN) == HIGH);
    
    if (btnState && !lastBtnState) {
      lcdPage = (lcdPage + 1) % 2;
      Serial.print("[Display Task] Switched to Page "); Serial.println(lcdPage);
      
      // Clear LCD screen immediately
      if (xSemaphoreTake(i2cMutex, portMAX_DELAY) == pdTRUE) {
        lcd.clear();
        xSemaphoreGive(i2cMutex);
      }
      vTaskDelay(pdMS_TO_TICKS(150)); // Debounce delay
    }
    lastBtnState = btnState;
    
    // Read variables and update LCD screen
    float t_dht = 0, h_dht = 0, t_bmp = 0, p_bmp = 0;
    
    if (xSemaphoreTake(i2cMutex, portMAX_DELAY) == pdTRUE) {
      t_dht = dhtTemp;
      h_dht = dhtHum;
      t_bmp = bmpTemp;
      p_bmp = bmpPress;
      
      // Update screen
      if (lcdPage == 0) {
        lcd.setCursor(0, 0);
        lcd.print("DHT Temp: "); lcd.print(t_dht, 1); lcd.print("C ");
        lcd.setCursor(0, 1);
        lcd.print("DHT Hum:  "); lcd.print(h_dht, 0); lcd.print("% ");
      } else {
        lcd.setCursor(0, 0);
        lcd.print("BMP Temp: "); lcd.print(t_bmp, 1); lcd.print("C ");
        lcd.setCursor(0, 1);
        lcd.print("Press: "); lcd.print(p_bmp, 1); lcd.print("hPa");
      }
      
      xSemaphoreGive(i2cMutex);
    }
    
    vTaskDelay(pdMS_TO_TICKS(200)); // Update LCD at 5Hz
  }
}

void setup() {
  Serial.begin(115200);
  Wire.begin();
  
  // Initialize LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Env HUD Scheduler");
  delay(1500);
  lcd.clear();
  
  // Create Mutex
  i2cMutex = xSemaphoreCreateMutex();
  
  if (i2cMutex != NULL) {
    // Create Tasks
    xTaskCreate(TaskReadDHT, "DHT Reader", 2048, NULL, 1, NULL);
    xTaskCreate(TaskReadBMP, "BMP Reader", 2048, NULL, 1, NULL);
    xTaskCreate(TaskDisplay, "LCD Display", 2048, NULL, 2, NULL); // Higher priority
  } else {
    Serial.println("Mutex creation failed!");
    while(1) {}
  }
}

void loop() {
  // Empty loop
  vTaskDelay(pdMS_TO_TICKS(1000));
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, **BMP180**, **Button**, and **16x2 I2C LCD** onto the canvas.
2. Wire Button to **GPIO12**, DHT22 to **GPIO4**, and I2C devices to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Monitor the LCD screen. Adjust DHT22 values. Watch Row 0/1 update.
5. Click the button widget. Watch the LCD switch to Page 1, displaying BMP180 pressure/temp data.

## Expected Output
Serial Monitor:
```
[DHT Task] Updated readings: 24.5C
[BMP Task] Updated readings: 1013.2hPa
[Display Task] Switched to Page 1
```

LCD Display (Page 0):
```
DHT Temp: 24.5C
DHT Hum:  45%
```

LCD Display (Page 1):
```
BMP Temp: 24.2C
Press: 1013.2hPa
```

## Expected Canvas Behavior
* The LCD displays the live DHT22 temperature and humidity values.
* Clicking the button widget clears the LCD and displays the live BMP180 values.
* The system is stable and does not lock up when both sensors are updating.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `xSemaphoreTake(i2cMutex, ...)` | Locks the I2C bus before reading or writing to prevent conflicts between the sensors and the LCD. |
| `lcdPage = (lcdPage + 1) % 2` | Toggles the active page index when the button is clicked. |
| `vTaskDelay(pdMS_TO_TICKS(2000))` | Schedules the DHT task to run once every 2 seconds, freeing the CPU in the meantime. |

## Hardware & Safety Concept: Inter-Task Resource Synchronization
In simple projects, all code runs in a single loop, so conflicts are avoided naturally. In an RTOS-based system, tasks run in parallel and can interrupt each other. If the DHT task interrupts the LCD task while the latter is updating the screen, the I2C signals will collide, corrupting the display. Using a **Mutex** ensures that only one task can access the shared I2C bus at a time, protecting signal integrity.

## Try This! (Challenges)
1. **Dynamic warning buzzer**: Add a buzzer on GPIO 13. If either the DHT or BMP temperature exceeds 35.0 °C, sound the buzzer.
2. **Auto-cycle pages**: Implement a timer that automatically switches pages every 5 seconds if the button is not pressed.
3. **Sensor offline recovery**: Display a warning screen on the LCD if a sensor fails to respond for more than three cycles.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD display freezes | Mutex deadlock | Verify that every `xSemaphoreTake` call has a matching `xSemaphoreGive` |
| DHT22 readings are static | Data pin conflict | Verify that the DHT22 data pin is wired to GPIO 4 |
| The ESP32 crashes on startup | Stack size too small | Increase task stack allocations in `xTaskCreate` to 2048 or 3072 bytes |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [139 - ESP32 Multi-Sensor Environmental HUD](139-esp32-multi-sensor-environmental-hud.md)
- [172 - ESP32 RTOS Mutex Shared Resource Protector](172-esp32-rtos-mutex-shared-resource-protector.md)
- [171 - ESP32 Multi-tasking RTOS Scheduler Demo](171-esp32-multi-tasking-rtos-scheduler-demo.md)
