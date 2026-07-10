# 183 - Pico Multithreaded Communication Pipes

Build a message-passing telemetry system that implements a thread-safe Queue class to stream sensor packages from a producer core to a consumer core.

## Goal
Learn how to implement a thread-safe Queue structure using list operations and mutex locks to establish clean communication pipes between independent processor cores in MicroPython.

## What You Will Build
A producer-consumer data processor:
- **Core 1 (Producer Thread)**: Periodically reads a DHT22 Climate Sensor (GP12), packages the data, and pushes it into a shared, thread-safe queue.
- **Core 0 (Consumer Thread)**: Continuously polls the queue. When data arrives, it pulls the package and displays the values on an SSD1306 OLED screen (GP4, GP5).
- **Control Button (GP13)**: Sends a trigger event from Core 0 back to Core 1 to request an immediate, out-of-band sensor reading.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | DATA | GP12 | Yellow | Climate sensor data |
| DHT22 | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| Button | Terminal 1 | GP13 | White | Instant sample trigger |
| Button | Terminal 2 | GND | Black | Ground return |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the DHT22 to GP12, the trigger button to GP13, and the OLED screen to GP4/GP5. All modules must share a common ground.

## Code
```python
import _thread
from machine import Pin, I2C
import utime, dht, ssd1306

# Sensors & Displays
dht_sensor = dht.DHT22(Pin(12))
btn_trigger = Pin(13, Pin.IN, Pin.PULL_UP)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# --- Thread-Safe Queue Class ---
class ThreadSafeQueue:
    def __init__(self):
        self.queue = []
        self.lock = _thread.allocate_lock()

    def put(self, item):
        """Append an item to the end of the queue safely."""
        self.lock.acquire()
        self.queue.append(item)
        self.lock.release()

    def get(self):
        """Remove and return the oldest item from the queue safely. Returns None if empty."""
        self.lock.acquire()
        item = None
        if len(self.queue) > 0:
            item = self.queue.pop(0)
        self.lock.release()
        return item

    def size(self):
        self.lock.acquire()
        length = len(self.queue)
        self.lock.release()
        return length

# Shared communication pipes
sensor_queue = ThreadSafeQueue()
trigger_flag = False
flag_lock = _thread.allocate_lock()

def core1_producer():
    """Producer task running on Core 1."""
    global trigger_flag
    print("[Core 1] Producer thread started.")
    
    last_sample = 0
    while True:
        now = utime.ticks_ms()
        
        # Check if Core 0 requested an immediate sample
        flag_lock.acquire()
        force_sample = trigger_flag
        trigger_flag = False
        flag_lock.release()
        
        # Sample if 2 seconds have elapsed OR if forced by button
        if utime.ticks_diff(now, last_sample) >= 2000 or force_sample:
            last_sample = now
            try:
                dht_sensor.measure()
                t = dht_sensor.temperature()
                h = dht_sensor.humidity()
                
                # Push data tuple (temp, hum, type) into queue
                source = "MANUAL" if force_sample else "AUTO  "
                sensor_queue.put((t, h, source))
                print("[Core 1] Produced data package to queue.")
            except OSError:
                print("[Core 1] Sensor read failed.")
                
        utime.sleep_ms(50)

# Start Core 1 background thread
_thread.start_new_thread(core1_producer, ())

# Global variables on Core 0 to store last pulled values
current_t = 0.0
current_h = 0.0
current_src = "INIT"
queue_reads = 0

oled.fill(0)
oled.text("Pipe Logger Init", 4, 28)
oled.show()
utime.sleep(1.0)

print("[Core 0] Consumer active. Waiting for data.")

while True:
    # 1. Check for button trigger to request data
    if btn_trigger.value() == 0:
        flag_lock.acquire()
        trigger_flag = True
        flag_lock.release()
        print("[Core 0] Requested manual sample.")
        utime.sleep_ms(200)  # Simple debounce
        while btn_trigger.value() == 0:
            utime.sleep_ms(10)
            
    # 2. Check for incoming data in the queue (Consume)
    package = sensor_queue.get()
    if package is not None:
        current_t, current_h, current_src = package
        queue_reads += 1
        print("[Core 0] Consumed package: T:{:.1f} H:{:.0f} S:{}".format(
            current_t, current_h, current_src))
        
    # 3. Update OLED Display
    oled.fill(0)
    oled.rect(0, 0, 128, 14, 1)
    oled.text("COMMUNICATION PIPE", 2, 3, 0)
    
    oled.text("Temp   : {:.1f} C".format(current_t), 10, 20, 1)
    oled.text("Humid  : {:.1f} %".format(current_h), 10, 32, 1)
    oled.text("Source : {}".format(current_src), 10, 44, 1)
    oled.text("Pkgs   : {} (Q:{})".format(queue_reads, sensor_queue.size()), 10, 56, 1)
    
    oled.rect(0, 0, 128, 64, 1)
    oled.show()
    
    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **Push Button**, and **SSD1306 OLED** onto the canvas.
2. Connect DHT22 to **GP12**, Button to **GP13**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the package counter `Pkgs` increment every 2 seconds. Click the Button on **GP13**. Verify that the source changes to `MANUAL` and the package count increments immediately.

## Expected Output
Terminal:
```
[Core 1] Producer thread started.
[Core 0] Consumer active. Waiting for data.
[Core 1] Produced data package to queue.
[Core 0] Consumed package: T:24.0 H:50 S:AUTO  
[Core 0] Requested manual sample.
[Core 1] Produced data package to queue.
[Core 0] Consumed package: T:24.0 H:50 S:MANUAL
```

## Expected Canvas Behavior
* Boot: OLED shows `Pipe Logger Init`.
* Every 2 seconds: OLED updates the displayed values, and `Pkgs` increments by 1.
* Press Button (GP13): OLED updates immediately, `Source` reads `MANUAL`, and `Pkgs` increments instantly.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `sensor_queue.put(...)` | Pushes a data package onto the queue safely from the producer thread. |
| `sensor_queue.get()` | Safely pops the oldest data package from the queue for processing in the consumer thread. |

## Hardware & Safety Concept: Producer-Consumer Decoupling
In real-time systems, the **Producer** (e.g. high-speed ADCs or network sockets) and the **Consumer** (e.g. slow flash writers or graphical displays) run at different speeds. Direct coupling forces the producer to slow down to match the consumer's speed, risking data loss. Queue-based communication pipes buffer data packages, decoupling the two cores and ensuring no data is dropped during consumer operations.

## Try This! (Challenges)
1. **Queue Overflow Alarm**: Sound a buzzer on GP14 if the queue size exceeds 5 (indicating the consumer is running too slowly).
2. **Dynamic Log Interval**: Add a second button on GP15 that toggles the default producer logging interval between 2, 5, and 10 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| App hangs or reports memory errors | Thread queue grows indefinitely | Ensure the consumer thread is pulling packages out of the queue using `get()`. If the consumer stalls, the queue will fill RAM and crash the system. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [120 - Pico Bluetooth DHT22 Telemetry Broadcast](../intermediate/120-pico-bluetooth-dht22-telemetry-broadcast.md)
- [181 - Pico Multithreaded Core Execution](181-pico-multithreaded-core-execution.md)
