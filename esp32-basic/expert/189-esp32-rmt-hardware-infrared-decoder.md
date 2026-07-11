# 189 - ESP32 RMT Hardware Infrared Decoder

Build a high-precision infrared remote control receiver on the ESP32 that configures the internal Remote Control (RMT) hardware peripheral to capture and measure IR pulse durations, print timing logs, and flash an LED when a signal is received.

## Goal
Learn how to configure the ESP32's built-in hardware RMT logic analyzer peripheral, capture sub-millisecond signal timings, and parse raw pulse trains.

## What You Will Build
An IR receiver module (e.g. VS1838B) is connected to GPIO 4. A Green LED is on GPIO 12. The ESP32 does not use standard CPU interrupts or polling to read the IR pin. Instead, the hardware RMT module measures the durations of incoming high and low pulses and stores them in a memory buffer. When a transmission ends, the ESP32 logs the timings and flashes the LED.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| IR Receiver Module (VS1838B) | `ir_receiver` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Receiver | OUT (Signal) | GPIO4 | Yellow | RMT Input pin |
| IR Receiver | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | Decoded confirmation LED |
| Common ground| GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the IR receiver signal pin to GPIO 4. Power the receiver from 3.3V.

## Code
```cpp
// RMT Hardware Infrared Decoder (Precise Timing Logs)
#include <Arduino.h>
#include "driver/rmt.h"

const int LED_PIN = 12;
#define RMT_RX_PIN 4 // Pin connected to IR receiver OUT

#define RMT_RX_CHANNEL RMT_CHANNEL_0
#define BUFFER_SIZE    100

// Buffer to receive RMT hardware items
RingbufHandle_t rb = NULL;

void initRMTRx() {
  Serial.println("Configuring RMT receiver peripheral...");
  
  // 1. Configure RMT Receiver parameters
  rmt_config_t rmt_rx_config;
  rmt_rx_config.rmt_mode = RMT_MODE_RX; // Receive mode
  rmt_rx_config.channel = RMT_RX_CHANNEL;
  rmt_rx_config.gpio_num = (gpio_num_t)RMT_RX_PIN;
  
  // Clock divider: 80. Base clock = 80 MHz -> 1 MHz tick rate (1 tick = 1 microsecond)
  rmt_rx_config.clk_div = 80; 
  
  // Memory block allocation (1 block = 64 items)
  rmt_rx_config.mem_block_num = 1; 
  
  // Configure filter to ignore pulses shorter than 100 microseconds (noise filter)
  rmt_rx_config.rx_config.filter_en = true;
  rmt_rx_config.rx_config.filter_ticks_thresh = 100;
  
  // Idle threshold: if the line stays in one state for 12,000 ticks (12 ms),
  // the transmission is considered complete
  rmt_rx_config.rx_config.idle_threshold = 12000; 

  // 2. Initialize RMT channel
  rmt_config(&rmt_rx_config);
  
  // 3. Install RMT driver and allocate ring buffer
  rmt_driver_install(RMT_RX_CHANNEL, 1000, 0);
  
  // Retrieve a handle to the driver's internal ring buffer
  rmt_get_ringbuf_handle(RMT_RX_CHANNEL, &rb);
  
  // 4. Start RMT receiver
  rmt_rx_start(RMT_RX_CHANNEL, true);
  
  Serial.println("RMT receiver active. Waiting for IR signals.");
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  initRMTRx();
}

void loop() {
  size_t rx_size = 0;
  
  // 5. Check if data is available in the ring buffer
  // Wait up to 100 ms for data
  rmt_item32_t* item = (rmt_item32_t*) xRingbufferReceive(rb, &rx_size, pdMS_TO_TICKS(100));
  
  if (item != NULL) {
    // Flash Green LED to indicate transmission received
    digitalWrite(LED_PIN, HIGH);
    
    // Process items (each item holds high/low pulse durations)
    int num_items = rx_size / sizeof(rmt_item32_t);
    Serial.print("\n--- Received IR Frame: "); Serial.print(num_items); Serial.println(" items ---");
    
    for (int i = 0; i < num_items; i++) {
      // duration0 = low/high duration, level0 = high/low level (0 or 1)
      uint32_t dur0 = item[i].duration0;
      uint32_t level0 = item[i].level0;
      uint32_t dur1 = item[i].duration1;
      uint32_t level1 = item[i].level1;
      
      // Print pulse timings
      Serial.print("Pulse "); Serial.print(i);
      Serial.print(": [Lvl "); Serial.print(level0); Serial.print(" | "); Serial.print(dur0); Serial.print(" us] ");
      Serial.print("[Lvl "); Serial.print(level1); Serial.print(" | "); Serial.print(dur1); Serial.println(" us]");
    }
    
    // Release the buffer memory
    vRingbufferReturnItem(rb, (void*)item);
    
    delay(50); // Small cooldown
    digitalWrite(LED_PIN, LOW);
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **IR Receiver**, and **LED** onto the canvas.
2. Wire IR Receiver OUT to **GPIO4** and LED to **GPIO12**.
3. Paste the code and click **Run**.
4. Click buttons on the simulated IR remote widget. Watch the Green LED flash.
5. Inspect the Serial Monitor. Watch the captured pulse durations log in microseconds.

## Expected Output
Serial Monitor:
```
RMT receiver active. Waiting for IR signals.

--- Received IR Frame: 33 items ---
Pulse 0: [Lvl 0 | 9000 us] [Lvl 1 | 4500 us]
Pulse 1: [Lvl 0 | 560 us] [Lvl 1 | 560 us]
Pulse 2: [Lvl 0 | 560 us] [Lvl 1 | 1680 us]
```

## Expected Canvas Behavior
* At boot, the Green LED is OFF.
* Clicking any button on the simulated IR remote flashes the Green LED once.
* The Serial Monitor logs the raw timing codes of the pressed button in microseconds.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `rmt_rx_config.clk_div = 80` | Sets the RMT clock rate to 1 MHz, providing 1-microsecond measurement resolution. |
| `rx_config.idle_threshold` | Tells the RMT module to stop capturing and complete the frame if the line is idle for 12 ms. |
| `xRingbufferReceive(...)` | Retrieves the captured pulse data from the RMT driver's ring buffer. |

## Hardware & Safety Concept: The ESP32 RMT (Remote Control) Peripheral
Decoding infrared remote signals requires measuring rapid pulse transitions (typically 500 us to 2 ms long). If the CPU is busy writing to an SD card or updating an OLED screen, it will miss these transitions, causing decoding errors. The ESP32 contains a hardware **RMT (Remote Control)** peripheral. The RMT module acts as a hardware logic analyzer: it automatically measures and records the durations of incoming pulses to a memory buffer in the background, ensuring accurate decoding.

## Try This! (Challenges)
1. **NEC Protocol Decoder**: Write code to analyze the header pulse (9 ms LOW, 4.5 ms HIGH) and bit timings to decode remote buttons into hex codes.
2. **Command Action Toggle**: Toggle a relay (GPIO 13) when a specific remote button command is decoded.
3. **IR Transmitter**: Use a second RMT channel to send IR remote commands via an IR LED.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The log prints 0 items | Pin mapping conflict | Verify that the IR receiver OUT pin is connected to GPIO 4 |
| Captured timings are unstable | Glitch filter threshold too low | Increase the filter threshold `filter_ticks_thresh` to ignore noise spikes |
| The ESP32 crashes on receive | Memory buffer allocation failed | Ensure the ring buffer is allocated using `rmt_driver_install` before starting |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [117 - ESP32 IR Remote Receiver Command Decoder](../intermediate/117-esp32-ir-remote-receiver-command-decoder.md)
- [118 - ESP32 IR Remote LED Control](../intermediate/118-esp32-ir-remote-led-control.md)
- [190 - ESP32 RMT Hardware NeoPixel Driver](190-esp32-rmt-hardware-neopixel-driver.md) (Next project)
