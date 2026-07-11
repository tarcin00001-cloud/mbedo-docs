# 179 - ESP32 Direct Register Port Manipulation

Build a high-speed GPIO controller on the ESP32 that configures pin directions, reads inputs, and toggles outputs by writing directly to the ESP32 GPIO peripheral registers, comparing execution speeds against standard Arduino functions.

## Goal
Learn how to read and write directly to ESP32 hardware registers (W1TS, W1TC, ENABLE, IN), bypass standard Arduino overhead, and toggle pins at megahertz frequencies.

## What You Will Build
A Green LED is on GPIO 12, a Red LED on GPIO 13, and a push button on GPIO 4. The ESP32 does not use `digitalWrite`, `digitalRead`, or `pinMode`. Instead, it configures pin directions and sets/clears outputs by writing directly to registers. When the button is pressed, the code runs a high-speed toggle loop, comparing its speed to standard functions.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| 330 Ω Resistors (2) | `resistor` | No | Yes |
| 10 kΩ Resistor (1) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | High-speed toggle target |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Standard toggle comparison |
| Push Button | Pin 1 / Pin 2 | GPIO4 / 3V3 | Orange / Red | Loop switch input (active-HIGH) |
| Push Button | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Common ground| GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the button with an external 10 kΩ pull-down resistor to GPIO 4. Ground both LED cathodes.

## Code
```cpp
// Direct Register Port Manipulation (Direct GPIO Register writes)
#include <Arduino.h>
#include "soc/gpio_reg.h"

// Define pin bitmasks
// GPIO 12 bitmask (1 << 12) = 0x1000
const uint32_t MASK_LED_GREEN = (1 << 12); 
// GPIO 13 bitmask (1 << 13) = 0x2000
const uint32_t MASK_LED_RED   = (1 << 13); 
// GPIO 4 bitmask (1 << 4) = 0x0010
const uint32_t MASK_BUTTON    = (1 << 4);  

void setup() {
  Serial.begin(115200);
  delay(500);
  
  Serial.println("Configuring GPIO pins using registers...");
  
  // 1. Configure Pin Directions
  // Set GPIO 12 and 13 as outputs by setting their bits in the GPIO Enable Register
  uint32_t currentEnable = REG_READ(GPIO_ENABLE_REG);
  REG_WRITE(GPIO_ENABLE_REG, currentEnable | MASK_LED_GREEN | MASK_LED_RED);
  
  // Set GPIO 4 as input by clearing its bit in the GPIO Enable Register
  REG_WRITE(GPIO_ENABLE_REG, REG_READ(GPIO_ENABLE_REG) & ~MASK_BUTTON);
  
  // Configure pads for GPIO use (enable digital inputs/outputs)
  pinMode(12, OUTPUT);
  pinMode(13, OUTPUT);
  pinMode(4, INPUT);
  
  Serial.println("Direct Register setup complete.");
}

void runSpeedBenchmark() {
  unsigned long start, end;
  const int ITERATIONS = 100000;
  
  Serial.println("\n--- RUNNING SPEED BENCHMARK (100,000 toggles) ---");
  
  // Benchmark 1: Standard Arduino digitalWrite
  start = micros();
  for (int i = 0; i < ITERATIONS; i++) {
    digitalWrite(12, HIGH);
    digitalWrite(12, LOW);
  }
  end = micros();
  unsigned long timeDigitalWrite = end - start;
  Serial.print("Standard digitalWrite time:  ");
  Serial.print(timeDigitalWrite); Serial.println(" us");
  
  // Benchmark 2: Direct Register writes
  start = micros();
  for (int i = 0; i < ITERATIONS; i++) {
    // Write 1 to W1TS (Write 1 to Set) to set pin HIGH
    REG_WRITE(GPIO_OUT_W1TS_REG, MASK_LED_GREEN);
    // Write 1 to W1TC (Write 1 to Clear) to set pin LOW
    REG_WRITE(GPIO_OUT_W1TC_REG, MASK_LED_GREEN);
  }
  end = micros();
  unsigned long timeRegisterWrite = end - start;
  Serial.print("Direct Register Write time:  ");
  Serial.print(timeRegisterWrite); Serial.println(" us");
  
  // Calculate speed improvement factor
  float factor = (float)timeDigitalWrite / timeRegisterWrite;
  Serial.print("Speed Improvement Factor:   ");
  Serial.print(factor, 1); Serial.println("x faster!");
}

void loop() {
  // 2. Read input pin using registers
  // Read the GPIO Input Register and check if the button bit is set
  bool buttonPressed = (REG_READ(GPIO_IN_REG) & MASK_BUTTON) != 0;
  
  if (buttonPressed) {
    // Run speed benchmark
    runSpeedBenchmark();
    
    // Cool down delay to prevent repeating
    delay(2000); 
  } else {
    // 3. Normal Operation Mode (Slow Blink using registers)
    // Turn Green LED ON
    REG_WRITE(GPIO_OUT_W1TS_REG, MASK_LED_GREEN);
    // Turn Red LED OFF
    REG_WRITE(GPIO_OUT_W1TC_REG, MASK_LED_RED);
    delay(500);
    
    // Turn Green LED OFF
    REG_WRITE(GPIO_OUT_W1TC_REG, MASK_LED_GREEN);
    // Turn Red LED ON
    REG_WRITE(GPIO_OUT_W1TS_REG, MASK_LED_RED);
    delay(500);
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Button**, and two **LEDs** onto the canvas.
2. Wire Button to **GPIO4**, Green LED to **GPIO12**, and Red LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Watch the LEDs blink alternately. The system is using registers to control the pins.
5. Click the button widget. Watch the Serial Monitor print the speed benchmark results, showing the register writes speed improvement.

## Expected Output
Serial Monitor:
```
Configuring GPIO pins using registers...
Direct Register setup complete.

--- RUNNING SPEED BENCHMARK (100,000 toggles) ---
Standard digitalWrite time:  156200 us
Direct Register Write time:  12800 us
Speed Improvement Factor:   12.2x faster!
```

## Expected Canvas Behavior
* At boot, the Green and Red LEDs flash alternately every 500 ms.
* Clicking the button widget runs the benchmark. The Green LED flashes at a high frequency (appearing dim or solid to the eye) for a split second.
* The Serial Monitor prints the speed comparison.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `GPIO_ENABLE_REG` | The hardware register that configures the direction (input/output) of GPIO pins. |
| `GPIO_OUT_W1TS_REG` | **Write 1 to Set**: Writing a bit mask to this register sets the corresponding GPIO pins HIGH instantly. |
| `GPIO_OUT_W1TC_REG` | **Write 1 to Clear**: Writing a bit mask to this register sets the corresponding GPIO pins LOW instantly. |
| `GPIO_IN_REG` | The hardware register that reads the input states of all GPIO pins simultaneously. |

## Hardware & Safety Concept: Register-level access vs Arduino HAL
Standard Arduino functions like `digitalWrite` use a **Hardware Abstraction Layer (HAL)**. When you call `digitalWrite(12, HIGH)`, the CPU executes several instructions: looking up the pin mapping table, checking if the pin supports PWM, verifying interrupts, and checking boundaries. While this abstraction makes coding easier, it adds latency. Writing directly to **GPIO Registers** updates the hardware pin states in a single CPU cycle, which is essential for high-speed protocols (like custom software SPI or bit-banging displays).

## Try This! (Challenges)
1. **Simultaneous Multi-Pin Toggle**: Modify the code to toggle *both* the Green and Red LEDs at the exact same instant using a single combined register write.
2. **Frequency Counter**: Run the register loop continuously and toggle a pin, measuring the maximum output frequency on an oscilloscope.
3. **Register-level interrupt**: Configure a hardware interrupt (GPIO 4) using the GPIO interrupt registers.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The LEDs do not light up at all | Pad matrix not configured | Ensure that `pinMode` is called to configure the pad connection matrix to link the CPU core to the physical pin |
| Pins on GPIO 32-39 do not toggle | Input-only pins | Pins 34–39 are input-only pins and cannot be configured as outputs |
| Toggling registers causes crash | Out-of-bounds register write | Verify that only valid GPIO register addresses are written to |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [180 - ESP32 High-speed SPI DMA OLED Frame buffer](180-esp32-high-speed-spi-dma-oled-frame-buffer.md) (Next project)
- [181 - ESP32 Custom Hardware Timer Interrupts](181-esp32-custom-hardware-timer-interrupts.md)
- [53 - ESP32 Variable LED Blink Rates](../intermediate/53-esp32-variable-led-blink-rates.md)
