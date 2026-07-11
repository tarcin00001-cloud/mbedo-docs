# 173 - ESP32 RTOS Queue Data Producer-Consumer

Build an RTOS application on the ESP32 that implements a Producer-Consumer pattern: Task 1 (Producer) reads a DHT22 sensor and sends data packets into a FreeRTOS Queue, while Task 2 (Consumer) retrieves the packets from the queue, logs them, and triggers a buzzer alarm if the temperature is high.

## Goal
Learn how to create FreeRTOS queues, pass structured data packets between threads, and coordinate producer and consumer tasks.

## What You Will Build
A DHT22 sensor is connected to GPIO 4, and an active buzzer is on GPIO 13. Task 1 (Producer) samples temperature and humidity every 1 second, packs them into a custom C++ structure, and pushes the structure into a FreeRTOS queue. Task 2 (Consumer) waits for data packets to arrive in the queue, processes them, and activates the buzzer if the temperature exceeds 30.0 °C.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Temperature input |
| Active Buzzer | VCC (+) | GPIO13 | Blue | Alarm output |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the common ground between the buzzer and the ESP32. Wire the active buzzer control pin directly to GPIO 13.

## Code
```cpp
// RTOS Queue Data Producer-Consumer (Inter-task queues)
#include <Arduino.h>
#include <DHT.h>

const int DHT_PIN = 4;
const int BUZZER_PIN = 13;

#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

// Define structured message to pass through queue
struct SensorData {
  float temperature;
  float humidity;
  unsigned long timestamp;
};

// Queue Handle
QueueHandle_t sensorQueue;

// Task 1: Producer - Reads sensor and sends data to queue
void TaskProducer(void *pvParameters) {
  dht.begin();
  
  while (1) {
    SensorData data;
    data.temperature = dht.readTemperature();
    data.humidity = dht.readHumidity();
    data.timestamp = millis();
    
    if (!isnan(data.temperature) && !isnan(data.humidity)) {
      // Send structure to queue (wait 0 ticks if full)
      BaseType_t status = xQueueSend(sensorQueue, &data, 0);
      
      if (status == pdPASS) {
        Serial.print("[Producer] Sent: ");
        Serial.print(data.temperature, 1);
        Serial.println("C to queue.");
      } else {
        Serial.println("[Producer] Queue FULL. Dropped data.");
      }
    } else {
      Serial.println("[Producer] DHT sensor read error.");
    }
    
    // Sample once per second
    vTaskDelay(pdMS_TO_TICKS(1000)); 
  }
}

// Task 2: Consumer - Reads from queue, logs, and checks thresholds
void TaskConsumer(void *pvParameters) {
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  SensorData receivedData;
  
  while (1) {
    // Wait indefinitely (portMAX_DELAY) until a data packet is available in the queue
    if (xQueueReceive(sensorQueue, &receivedData, portMAX_DELAY) == pdTRUE) {
      Serial.print("[Consumer] Received: ");
      Serial.print(receivedData.temperature, 1);
      Serial.print("C | Hum: ");
      Serial.print(receivedData.humidity, 0);
      Serial.print("% | Time: ");
      Serial.print(receivedData.timestamp);
      Serial.println(" ms");
      
      // Threshold check: turn ON buzzer if temperature > 30C
      if (receivedData.temperature > 30.0) {
        Serial.println("[Consumer] !!! HIGH TEMP WARNING !!!");
        digitalWrite(BUZZER_PIN, HIGH);
        vTaskDelay(pdMS_TO_TICKS(200)); // beep briefly
        digitalWrite(BUZZER_PIN, LOW);
      }
    }
  }
}

void setup() {
  Serial.begin(115200);
  Serial.println("Starting RTOS Producer-Consumer setup...");
  
  // Create a queue to hold up to 5 SensorData structures
  sensorQueue = xQueueCreate(5, sizeof(SensorData));
  
  if (sensorQueue != NULL) {
    Serial.println("Sensor Queue created successfully.");
    
    // Create Producer Task (Priority 1)
    xTaskCreate(
      TaskProducer,
      "Producer Task",
      2048,
      NULL,
      1,
      NULL
    );
    
    // Create Consumer Task (Priority 2 - higher to ensure immediate processing)
    xTaskCreate(
      TaskConsumer,
      "Consumer Task",
      2048,
      NULL,
      2,
      NULL
    );
  } else {
    Serial.println("Error creating Sensor Queue!");
    while(1) {}
  }
}

void loop() {
  // Empty loop
  vTaskDelay(pdMS_TO_TICKS(1000));
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, and **Buzzer** onto the canvas.
2. Wire DHT22 to **GPIO4** and Buzzer to **GPIO13**.
3. Paste the code and click **Run**.
4. Monitor the Serial Monitor. Watch the Producer send data packets and the Consumer print them.
5. Slide the DHT22 temperature past 30 °C. Watch the buzzer beep.

## Expected Output
Serial Monitor:
```
Starting RTOS Producer-Consumer setup...
Sensor Queue created successfully.
[Producer] Sent: 28.5C to queue.
[Consumer] Received: 28.5C | Hum: 45% | Time: 1500 ms
[Producer] Sent: 31.2C to queue.
[Consumer] Received: 31.2C | Hum: 45% | Time: 2500 ms
[Consumer] !!! HIGH TEMP WARNING !!!
```

## Expected Canvas Behavior
* The Producer task sends data packets every 1 second.
* The Consumer task wakes up instantly when a packet is sent, logging it to the Serial Monitor.
* Adjusting the DHT22 temperature slider above 30.0 °C causes the buzzer widget to pulse green.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `xQueueCreate(5, ...)` | Allocates memory for a thread-safe FIFO queue holding up to 5 custom structures. |
| `xQueueSend(sensorQueue, ...)` | Pushes a copy of the structure into the queue. Returns `pdPASS` if successful. |
| `xQueueReceive(sensorQueue, ...)` | Blocks the task until a packet arrives in the queue, then pops it into `receivedData`. |

## Hardware & Safety Concept: Thread-Safe Queues vs Shared Memory
In a multi-tasking system, passing data between tasks using a simple global variable is unsafe. If the Producer is halfway through updating the variable (e.g. writing the first byte of a float) when the Consumer interrupts and reads it, the Consumer reads corrupt data. FreeRTOS **Queues** copy data thread-safely by blocking tasks and using internal locks, preventing race conditions.

## Try This! (Challenges)
1. **Queue Full Handler alert**: Lower the consumer priority to 1 and change its delay to 3 seconds. Watch the queue fill up and the Producer print warning messages.
2. **Multiple Producers**: Create a second producer task (potentiometer on GPIO 34) that sends values to the same queue.
3. **Queue Reset Button**: Add a button on GPIO 12 that resets and empties the queue when pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Output shows "Queue FULL" | Consumer task is blocked or running too slowly | Increase the consumer task's priority or reduce its processing delay |
| Consumer never receives data | Queue creation failed | Verify that `sensorQueue` is initialized in `setup()` before tasks start |
| ESP32 crashes on startup | Stack overflow | Increase the stack size in `xTaskCreate` to 2048 or 4096 bytes |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [171 - ESP32 Multi-tasking RTOS Scheduler Demo](171-esp32-multi-tasking-rtos-scheduler-demo.md)
- [172 - ESP32 RTOS Mutex Shared Resource Protector](172-esp32-rtos-mutex-shared-resource-protector.md)
- [174 - ESP32 RTOS Event Groups System](174-esp32-rtos-event-groups-system.md) (Next project)
