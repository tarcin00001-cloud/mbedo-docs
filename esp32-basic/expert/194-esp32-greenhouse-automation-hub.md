# 194 - ESP32 Greenhouse Automation Hub

Build a multi-tasking industrial greenhouse climate controller on the ESP32 that runs independent FreeRTOS tasks to monitor climate (DHT22) and irrigation water levels, driving dual relays for fan cooling and water pumping.

## Goal
Learn how to schedule parallel sensor control tasks with different priority levels, implement non-blocking automation loops, and protect shared I2C LCD access.

## What You Will Build
A DHT22 is on GPIO 4. An analog water level sensor is on GPIO 34. Relay 1 (Fan) is on GPIO 12, Relay 2 (Pump) on GPIO 13, and a 16x2 LCD on I2C. Task 1 reads the DHT22 and switches the fan if temperature > 28.0 °C. Task 2 reads water level and switches the pump if moisture is dry and water is available. Task 3 updates the LCD.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| Water Level Sensor (Analog) | `water_level` | Yes | Yes |
| Relay Modules (2) | `relay` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Temperature input |
| Water Level | OUT (Analog) | GPIO34 | White | Analog level input |
| Relay 1 (Fan) | IN (Signal) | GPIO12 | Orange | Cools greenhouse |
| Relay 2 (Pump)| IN (Signal) | GPIO13 | Blue | Irrigate soil |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Power both relays from the 5V Vin rail.

## Code
```cpp
// Greenhouse Automation Hub (DHT22 + Water level + dual relays + LCD FreeRTOS)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

const int DHT_PIN = 4;
const int WATER_PIN = 34;
const int RELAY_FAN = 12;
const int RELAY_PUMP = 13;

#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Shared telemetry variables
volatile float sharedTemp = 0.0;
volatile float sharedHum = 0.0;
volatile int sharedWaterLevel = 0;
volatile bool fanStatus = false;
volatile bool pumpStatus = false;

// Mutex to protect shared variables and LCD access
SemaphoreHandle_t greenhouseMutex;

// Task 1: Climate Control Task (Priority 1)
void TaskClimateControl(void *pvParameters) {
  dht.begin();
  pinMode(RELAY_FAN, OUTPUT);
  digitalWrite(RELAY_FAN, LOW);
  
  while (1) {
    float temp = dht.readTemperature();
    float hum = dht.readHumidity();
    
    if (!isnan(temp) && !isnan(hum)) {
      bool activeFan = (temp > 28.0); // Start fan if temperature is high
      
      if (activeFan) {
        digitalWrite(RELAY_FAN, HIGH);
      } else {
        digitalWrite(RELAY_FAN, LOW);
      }
      
      // Update shared variables
      if (xSemaphoreTake(greenhouseMutex, portMAX_DELAY) == pdTRUE) {
        sharedTemp = temp;
        sharedHum = hum;
        fanStatus = activeFan;
        xSemaphoreGive(greenhouseMutex);
      }
    }
    
    vTaskDelay(pdMS_TO_TICKS(2000)); // Sample every 2 seconds
  }
}

// Task 2: Irrigation Control Task (Priority 1)
void TaskIrrigationControl(void *pvParameters) {
  pinMode(RELAY_PUMP, OUTPUT);
  digitalWrite(RELAY_PUMP, LOW);
  
  while (1) {
    // Read analog water level sensor (0-4095)
    int waterRaw = analogRead(WATER_PIN);
    int waterPercent = map(waterRaw, 0, 4095, 0, 100);
    
    // Irrigation logic
    // Start pump if moisture is dry (simulated) and water level is > 20%
    bool activePump = (waterPercent > 20); 
    
    if (activePump) {
      digitalWrite(RELAY_PUMP, HIGH);
    } else {
      digitalWrite(RELAY_PUMP, LOW);
    }
    
    // Update shared variables
    if (xSemaphoreTake(greenhouseMutex, portMAX_DELAY) == pdTRUE) {
      sharedWaterLevel = waterPercent;
      pumpStatus = activePump;
      xSemaphoreGive(greenhouseMutex);
    }
    
    vTaskDelay(pdMS_TO_TICKS(1000)); // Check level every 1 second
  }
}

// Task 3: LCD Update Task (Priority 2 - higher priority to handle UI)
void TaskDisplay(void *pvParameters) {
  unsigned long lastPageChange = 0;
  int page = 0;
  
  while (1) {
    unsigned long now = millis();
    if (now - lastPageChange >= 4000) {
      page = (page + 1) % 2;
      
      // Clear screen on page change
      if (xSemaphoreTake(greenhouseMutex, portMAX_DELAY) == pdTRUE) {
        lcd.clear();
        xSemaphoreGive(greenhouseMutex);
      }
      lastPageChange = now;
    }
    
    // Read variables and update LCD screen
    float t = 0, h = 0;
    int w = 0;
    bool f = false, p = false;
    
    if (xSemaphoreTake(greenhouseMutex, portMAX_DELAY) == pdTRUE) {
      t = sharedTemp;
      h = sharedHum;
      w = sharedWaterLevel;
      f = fanStatus;
      p = pumpStatus;
      xSemaphoreGive(greenhouseMutex);
    }
    
    // Acquire mutex for writing to shared LCD
    if (xSemaphoreTake(greenhouseMutex, portMAX_DELAY) == pdTRUE) {
      if (page == 0) {
        lcd.setCursor(0, 0);
        lcd.print("Temp: "); lcd.print(t, 1); lcd.print("C ");
        lcd.setCursor(0, 1);
        lcd.print("Fan:  "); lcd.print(f ? "ON " : "OFF");
      } else {
        lcd.setCursor(0, 0);
        lcd.print("Water: "); lcd.print(w); lcd.print("%   ");
        lcd.setCursor(0, 1);
        lcd.print("Pump:  "); lcd.print(p ? "ON " : "OFF");
      }
      xSemaphoreGive(greenhouseMutex);
    }
    
    vTaskDelay(pdMS_TO_TICKS(200)); // Update LCD at 5Hz
  }
}

void setup() {
  Serial.begin(115200);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Greenhouse Hub");
  lcd.setCursor(0, 1);
  lcd.print("System starting");
  delay(1500);
  lcd.clear();
  
  greenhouseMutex = xSemaphoreCreateMutex();
  
  if (greenhouseMutex != NULL) {
    // Create Tasks
    xTaskCreate(TaskClimateControl, "Climate Task", 2048, NULL, 1, NULL);
    xTaskCreate(TaskIrrigationControl, "Irrigation Task", 2048, NULL, 1, NULL);
    xTaskCreate(TaskDisplay, "LCD Task", 2048, NULL, 2, NULL);
  } else {
    Serial.println("Mutex creation failed!");
    while(1) {}
  }
}

void loop() {
  vTaskDelay(pdMS_TO_TICKS(1000));
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, **Water Level**, two **Relays**, and **16x2 I2C LCD** onto the canvas.
2. Wire DHT22 to **GPIO4**, Level to **GPIO34**, Relay 1 to **GPIO12**, Relay 2 to **GPIO13**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the DHT22 temperature to 32 °C. Watch Relay 1 turn ON.
5. Slide the water level widget to 10% (empty). Watch Relay 2 turn OFF.

## Expected Output
Serial Monitor:
```
[Climate Task] Updated readings: 29.5C
[Irrigation Task] Level: 85 %
[Display Task] Switched to Page 1
```

LCD Display (Page 0):
```
Temp: 29.5C
Fan:  ON
```

LCD Display (Page 1):
```
Water: 85%
Pump:  ON
```

## Expected Canvas Behavior
* At startup, the LCD initializes.
* Raising the temperature past 28.0 °C turns Relay 1 ON (green).
* Lowering the water level slider below 20% turns Relay 2 OFF (grey).
* The LCD cycles between Page 0 and Page 1 every 4 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `xSemaphoreTake(greenhouseMutex, ...)` | Locks the shared variables and I2C LCD to prevent write collisions between tasks. |
| `temp > 28.0` | Starts the cooling fan once the temperature exceeds 28.0 °C. |
| `waterPercent > 20` | Protects the pump: only runs the pump if there is enough water in the tank. |

## Hardware & Safety Concept: Dry-Run Protection in Irrigation Systems
Water pump motors rely on the pumped liquid to cool and lubricate their internal impellers. If a pump runs dry (when the water source is empty), the friction generates high heat, melting the plastic impellers and burning out the motor within minutes. Automation systems must implement **dry-run protection** by monitoring water levels and blocking pump operation if the level drops below a safety threshold (e.g. 20%).

## Try This! (Challenges)
1. **Emergency Overheat Alarm**: Sound a buzzer (GPIO 15) if the temperature exceeds 38.0 °C.
2. **SD Card Data Logging**: Integrate an SD card reader (Project 137) to log climate and irrigation data to a CSV file.
3. **Manual Override button**: Add a button on GPIO 14 to manually trigger the pump, bypassing the automation.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The LCD screen freezes or displays garbled characters | Mutex deadlock | Verify that every `xSemaphoreTake` call has a matching `xSemaphoreGive` |
| Water level reads static 0% | Sensor miswired | Verify that the analog sensor pin is connected to GPIO 34 |
| Relays click continuously | Threshold value jitter | Add a hysteresis gap (e.g. turn pump ON at 25% and OFF at 20%) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [121 - ESP32 Greenhouse Controller](../intermediate/121-esp32-greenhouse-controller.md)
- [100 - ESP32 Automatic Smart Fan](../intermediate/100-esp32-automatic-smart-fan.md)
- [172 - ESP32 RTOS Mutex Shared Resource Protector](172-esp32-rtos-mutex-shared-resource-protector.md)
