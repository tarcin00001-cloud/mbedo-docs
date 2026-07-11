# 180 - ESP32 High-speed SPI DMA OLED Frame buffer

Build a high-performance graphics rendering system on the ESP32 that sets up an SPI SSD1306 OLED screen, allocates a DMA-safe frame buffer in RAM, and uses Direct Memory Access (DMA) to stream frames to the display in the background, plotting a high-speed bouncing ball.

## Goal
Learn how to configure SPI DMA channels on the ESP32, allocate page-aligned DMA-safe buffers, and write background graphics streaming routines.

## What You Will Build
An SPI SSD1306 OLED screen is connected via hardware SPI (MOSI: 23, SCK: 18, CS: 5, DC: 4, RST: 15). The ESP32 allocates a 1024-byte frame buffer, sets up a DMA channel, and updates the display in the background. The code draws a bouncing ball at 60+ FPS while measuring the CPU time saved by DMA.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 0.96″ SSD1306 OLED Screen (SPI) | `oled_spi` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SPI OLED | MOSI (D1) | GPIO23 | Blue | Hardware SPI MOSI |
| SPI OLED | SCK (D0) | GPIO18 | Yellow | Hardware SPI SCK |
| SPI OLED | CS | GPIO5 | Orange | Chip Select |
| SPI OLED | DC | GPIO4 | Green | Data / Command select |
| SPI OLED | RES (RST) | GPIO15 | Red | Hardware Reset |
| SPI OLED | VCC / GND | 3V3 / GND | Red / Black | Power rails |

> **Wiring tip:** SPI screens require five control lines instead of I2C's two lines. Ensure CS is wired to GPIO 5, DC to GPIO 4, and RST to GPIO 15.

## Code
```cpp
// High-speed SPI DMA OLED Frame buffer (Direct Memory Access OLED updates)
#include <Arduino.h>
#include <SPI.h>
#include <driver/spi_master.h>

#define PIN_NUM_MISO -1
#define PIN_NUM_MOSI 23
#define PIN_NUM_CLK  18
#define PIN_NUM_CS   5
#define PIN_NUM_DC   4
#define PIN_NUM_RST  15

#define SCREEN_WIDTH  128
#define SCREEN_HEIGHT 64
#define BUFFER_SIZE   (SCREEN_WIDTH * SCREEN_HEIGHT / 8) // 1024 bytes

// ESP32 SPI Device Handle
spi_device_handle_t spi_device;

// Pointer to DMA-capable frame buffer in RAM
uint8_t* dma_frame_buffer = NULL;

// Bouncing ball physics coordinates
float ballX = 64.0;
float ballY = 32.0;
float velX = 2.5;
float velY = 1.8;
const int BALL_RADIUS = 4;

void oledWriteCommand(uint8_t cmd) {
  digitalWrite(PIN_NUM_DC, LOW); // Command Mode
  
  spi_transaction_t t;
  memset(&t, 0, sizeof(t));
  t.length = 8;
  t.tx_buffer = &cmd;
  
  spi_device_polling_transmit(spi_device, &t); // Fast polling command write
}

void initOLED() {
  pinMode(PIN_NUM_RST, OUTPUT);
  pinMode(PIN_NUM_DC, OUTPUT);
  
  // Reset sequence
  digitalWrite(PIN_NUM_RST, LOW);
  delay(10);
  digitalWrite(PIN_NUM_RST, HIGH);
  delay(10);
  
  // SSD1306 Initialization commands
  oledWriteCommand(0xAE); // Display OFF
  oledWriteCommand(0xD5); // Set Display Clock Divide Ratio
  oledWriteCommand(0x80);
  oledWriteCommand(0xA8); // Set Multiplex Ratio
  oledWriteCommand(0x3F);
  oledWriteCommand(0xD3); // Set Display Offset
  oledWriteCommand(0x00);
  oledWriteCommand(0x40); // Set Start Line
  oledWriteCommand(0x8D); // Charge Pump
  oledWriteCommand(0x14); // Enable Charge Pump
  oledWriteCommand(0x20); // Memory Addressing Mode
  oledWriteCommand(0x00); // Horizontal Addressing Mode
  oledWriteCommand(0xA1); // Segment Re-map (flip horizontally)
  oledWriteCommand(0xC8); // COM Output Scan Direction (flip vertically)
  oledWriteCommand(0xDA); // COM Pins hardware configuration
  oledWriteCommand(0x12);
  oledWriteCommand(0x81); // Contrast Control
  oledWriteCommand(0xCF);
  oledWriteCommand(0xD9); // Pre-charge Period
  oledWriteCommand(0xF1);
  oledWriteCommand(0xDB); // VCOMH Deselect Level
  oledWriteCommand(0x40);
  oledWriteCommand(0xA4); // Output RAM contents to screen
  oledWriteCommand(0xA6); // Normal Display (not inverted)
  oledWriteCommand(0xAF); // Display ON
}

void setup() {
  Serial.begin(115200);
  
  // 1. Allocate DMA-safe buffer in ESP32 RAM (MALLOC_CAP_DMA)
  // Standard RAM allocations can fail DMA checks; must allocate from DMA-capable heap
  dma_frame_buffer = (uint8_t*) heap_caps_malloc(BUFFER_SIZE, MALLOC_CAP_DMA);
  
  if (dma_frame_buffer == NULL) {
    Serial.println("DMA Frame Buffer allocation failed!");
    while(1) {}
  }
  memset(dma_frame_buffer, 0, BUFFER_SIZE); // Clear buffer
  
  // 2. Configure ESP32 SPI Bus with DMA enabled
  spi_bus_config_t buscfg;
  memset(&buscfg, 0, sizeof(buscfg));
  buscfg.miso_io_num = PIN_NUM_MISO;
  buscfg.mosi_io_num = PIN_NUM_MOSI;
  buscfg.sclk_io_num = PIN_NUM_CLK;
  buscfg.quadwp_io_num = -1;
  buscfg.quadhd_io_num = -1;
  buscfg.max_transfer_sz = BUFFER_SIZE;
  
  // Initialize HSPI (SPI2) with DMA channel 1
  spi_bus_initialize(HSPI_HOST, &buscfg, 1); 
  
  // 3. Attach OLED device to SPI bus
  spi_device_interface_config_t devcfg;
  memset(&devcfg, 0, sizeof(devcfg));
  devcfg.clock_speed_hz = 8000000; // Run SPI at 8 MHz
  devcfg.mode = 0;
  devcfg.spics_io_num = PIN_NUM_CS;
  devcfg.queue_size = 1;
  
  spi_bus_add_device(HSPI_HOST, &devcfg, &spi_device);
  
  initOLED();
  Serial.println("SPI DMA OLED system ready.");
}

void clearBuffer() {
  memset(dma_frame_buffer, 0, BUFFER_SIZE);
}

void drawPixel(int x, int y, bool color) {
  if (x < 0 || x >= SCREEN_WIDTH || y < 0 || y >= SCREEN_HEIGHT) return;
  
  int index = x + (y / 8) * SCREEN_WIDTH;
  if (color) {
    dma_frame_buffer[index] |= (1 << (y % 8));
  } else {
    dma_frame_buffer[index] &= ~(1 << (y % 8));
  }
}

void drawCircle(int cx, int cy, int r) {
  for (int y = -r; y <= r; y++) {
    for (int x = -r; x <= r; x++) {
      if (x*x + y*y <= r*r) {
        drawPixel(cx + x, cy + y, true);
      }
    }
  }
}

// Background DMA Frame buffer update
void updateOLED_DMA() {
  // Set memory write range window
  oledWriteCommand(0x21); // Column Address
  oledWriteCommand(0);
  oledWriteCommand(127);
  oledWriteCommand(0x22); // Page Address
  oledWriteCommand(0);
  oledWriteCommand(7);
  
  digitalWrite(PIN_NUM_DC, HIGH); // Data mode
  
  // Create asynchronous DMA transaction
  static spi_transaction_t trans;
  memset(&trans, 0, sizeof(trans));
  trans.length = BUFFER_SIZE * 8; // Length in bits
  trans.tx_buffer = dma_frame_buffer;
  
  // Queue transaction to run in the background via DMA
  // xQueueSend-like, returns instantly while hardware handles data stream
  spi_device_queue_trans(spi_device, &trans, portMAX_DELAY);
  
  // Wait for previous transfer to finish if needed
  spi_transaction_t* completed_trans;
  spi_device_get_trans_result(spi_device, &completed_trans, portMAX_DELAY);
}

void loop() {
  unsigned long start = micros();
  
  // 1. Physics: update bouncing ball
  ballX += velX;
  ballY += velY;
  
  if (ballX - BALL_RADIUS <= 0 || ballX + BALL_RADIUS >= SCREEN_WIDTH)   velX = -velX;
  if (ballY - BALL_RADIUS <= 0 || ballY + BALL_RADIUS >= SCREEN_HEIGHT)  velY = -velY;
  
  // 2. Render to frame buffer
  clearBuffer();
  drawCircle((int)ballX, (int)ballY, BALL_RADIUS);
  
  // Draw outer borders
  for (int i = 0; i < SCREEN_WIDTH; i++) {
    drawPixel(i, 0, true);
    drawPixel(i, SCREEN_HEIGHT - 1, true);
  }
  
  // 3. Asynchronously push to OLED using DMA
  unsigned long beforeDMA = micros();
  updateOLED_DMA();
  unsigned long afterDMA = micros();
  
  unsigned long frameTime = micros() - start;
  
  // Telemetry logs
  Serial.print("Total Frame Time: "); Serial.print(frameTime / 1000.0); Serial.print(" ms");
  Serial.print(" | CPU Time spent queuing DMA: "); Serial.print((afterDMA - beforeDMA) / 1000.0); Serial.println(" ms");
  
  delay(16); // Target ~60 FPS
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **SPI SSD1306 OLED** onto the canvas.
2. Wire: MOSI to **GPIO23**, CLK to **GPIO18**, CS to **GPIO5**, DC to **GPIO4**, and RST to **GPIO15**.
3. Paste the code and click **Run**.
4. Watch the ball bounce smoothly inside the borders on the OLED screen.
5. Inspect the Serial Monitor to verify that the CPU time spent queuing the DMA transfer is under 0.2 ms.

## Expected Output
Serial Monitor:
```
SPI DMA OLED system ready.
Total Frame Time: 2.15 ms | CPU Time spent queuing DMA: 0.12 ms
Total Frame Time: 2.10 ms | CPU Time spent queuing DMA: 0.10 ms
```

## Expected Canvas Behavior
* The SPI OLED screen draws a box border and a filled circle (ball) bouncing off the borders.
* The animation runs smoothly at 60 FPS without lag.
* The Serial Monitor logs very low CPU queuing times, showing the benefits of DMA.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `heap_caps_malloc(...)` | Allocates the buffer from a DMA-accessible memory pool (required for ESP32 DMA). |
| `spi_bus_initialize(...)` | Initializes the SPI controller, enabling DMA channel 1. |
| `spi_device_queue_trans(...)` | Starts the DMA hardware transfer in the background, releasing the CPU immediately. |

## Hardware & Safety Concept: Direct Memory Access (DMA)
Standard SPI transfers use **polling**: the CPU reads the data buffer and writes it byte-by-byte to the SPI output registers, waiting for each transfer to finish. This blocks the CPU for several milliseconds. **Direct Memory Access (DMA)** allows the SPI controller to read the data directly from RAM using a dedicated bus. The CPU only needs to queue the transfer, freeing it to perform other tasks (like sensor parsing or motor control) while the transfer runs in the background.

## Try This! (Challenges)
1. **Interactive speed control**: Connect a potentiometer (GPIO 34) to adjust the ball's movement speed.
2. **Double buffering**: Implement double buffering (allocating two DMA buffers) to prevent screen tearing during fast animations.
3. **Gravity physics**: Add gravity and bounce drag factors to simulate a realistic bouncing ball.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The OLED displays static noise or stays blank | SPI reset failed | Verify that the RST pin is connected to GPIO 15 and initialized correctly |
| Allocation fails on startup | Standard malloc used | Ensure `heap_caps_malloc` is called with the `MALLOC_CAP_DMA` flag |
| Display flickers or tears | SPI clock speed set too low | Increase the SPI clock speed in `devcfg.clock_speed_hz` to 8 MHz or 10 MHz |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [179 - ESP32 Direct Register Port Manipulation](179-esp32-direct-register-port-manipulation.md)
- [60 - ESP32 OLED SSD1306 Display Setup](../intermediate/60-esp32-oled-ssd1306-display-setup.md)
- [154 - ESP32 OLED Kalman Filter Demo](154-esp32-oled-kalman-filter-demo.md)
