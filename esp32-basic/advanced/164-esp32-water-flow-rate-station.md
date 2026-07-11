# 164 - ESP32 Water Flow Rate Station

Build a fluid flow rate monitoring station that measures liquid flow using a Hall effect YF-S201 water flow sensor via GPIO interrupts, calculates real-time flow rate in Liters per minute, accumulates total volume consumed, and prints findings to an I2C LCD.

## Goal
Learn how to count high-speed sensor pulse streams using hardware interrupts, calculate fluid flow rates and cumulative volume integration, and display volumetric statistics.

## What You Will Build
A YF-S201 Hall effect water flow sensor is connected to GPIO 4 (interrupt pin). A 16x2 LCD is on I2C. The ESP32 uses an interrupt service routine (ISR) to count pulses, calculates the flow rate (L/min) and cumulative volume (Liters) every 1 second, and displays both values on the LCD.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| YF-S201 Water Flow Sensor | `flow_sensor` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Flow Sensor | OUT (Pulse) | GPIO4 | Yellow | Hall sensor pulse output |
| Flow Sensor | VCC / GND | 5V / GND | Red / Black | Sensor power |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |

> **Wiring tip:** The YF-S201 operates on 5V. Its pulse output requires a pull-up resistor (internal or external) to ensure clean digital signal transitions. Connect the sensor's pulse pin directly to GPIO 4.

## Code
```cpp
// Water Flow Rate Station (Flow sensor + LCD)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int FLOW_PIN = 4;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Pulse count from Hall sensor (declared volatile as it changes in ISR)
volatile int pulseCount = 0;

// Flow calibration factor
// YF-S201 output: F = 7.5 * Q (Q is flow rate in L/min, F is frequency in Hz)
const float CALIBRATION_FACTOR = 7.5; 

unsigned long lastTime = 0;
float flowRateLmin = 0.0;
float totalLiters = 0.0;

// Interrupt Service Routine (ISR) called on every sensor pulse
void IRAM_ATTR countFlowPulses() {
  pulseCount++;
}

void setup() {
  Serial.begin(115200);
  
  pinMode(FLOW_PIN, INPUT_PULLUP);
  
  // Attach interrupt to flow sensor pin (falling edge)
  attachInterrupt(digitalPinToInterrupt(FLOW_PIN), countFlowPulses, FALLING);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Flow Station");
  lcd.setCursor(0, 1);
  lcd.print("Initializing...");
  
  lastTime = millis();
  delay(1500);
  lcd.clear();
}

void loop() {
  unsigned long now = millis();
  
  // Calculate flow parameters every 1 second (1000 ms)
  if (now - lastTime >= 1000) {
    float dtSeconds = (now - lastTime) / 1000.0;
    
    // Disable interrupts briefly to read pulse count safely
    noInterrupts();
    int pulses = pulseCount;
    pulseCount = 0;
    interrupts();
    
    // Pulse frequency (Hz) = pulses / time
    float pulseFrequency = (float)pulses / dtSeconds;
    
    // Flow Rate (L/min) = Frequency / Calibration Factor
    flowRateLmin = pulseFrequency / CALIBRATION_FACTOR;
    
    // Flow Volume (Liters) = Flow Rate (L/min) * Time (minutes)
    // Time interval in minutes = dtSeconds / 60.0
    float volumeChange = flowRateLmin * (dtSeconds / 60.0);
    totalLiters += volumeChange;
    
    lastTime = now;
    
    // Update LCD Screen
    lcd.setCursor(0, 0);
    lcd.print("Flow: ");
    lcd.print(flowRateLmin, 1);
    lcd.print(" L/min   "); // Clear trailing space
    
    lcd.setCursor(0, 1);
    lcd.print("Total: ");
    lcd.print(totalLiters, 2);
    lcd.print(" L     ");
    
    Serial.print("Freq: "); Serial.print(pulseFrequency, 1);
    Serial.print(" Hz | Flow Rate: "); Serial.print(flowRateLmin, 2);
    Serial.print(" L/min | Volume: "); Serial.print(totalLiters, 3); Serial.println(" L");
  }
  
  delay(10); // Loop pace
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Water Flow Sensor**, and **16x2 I2C LCD** onto the canvas.
2. Wire Flow OUT to **GPIO4** and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Adjust the flow rate slider on the simulated sensor widget. Watch the L/min and total liters accumulate on the LCD.

## Expected Output
Serial Monitor:
```
Freq: 75.0 Hz | Flow Rate: 10.00 L/min | Volume: 0.167 L
Freq: 75.0 Hz | Flow Rate: 10.00 L/min | Volume: 0.333 L
```

LCD Display:
```
Flow: 10.0 L/min
Total: 0.33 L
```

## Expected Canvas Behavior
* Adjusting the flow rate slider on the simulated sensor sends a pulse train to the ESP32.
* The LCD displays the current flow rate and accumulated volume, updating every 1 second.
* Setting the slider to 0 stops the flow rate, leaving the accumulated volume total static.

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `countFlowPulses()` | ISR incrementing the pulse counter on every falling edge of the sensor pin. |
| `flowRateLmin = ...` | Computes flow rate by dividing pulse frequency by the calibration constant. |
| `totalLiters += volumeChange` | Integrates flow volume over time to calculate cumulative consumption. |

## Hardware & Safety Concept: Hall Effect Turbine Flow Sensors
A YF-S201 water flow sensor contains a pinwheel turbine that rotates as water passes through the sensor body. A small magnet is mounted on the turbine, and a Hall effect sensor measures the magnetic poles passing by, outputting a square wave pulse train. The frequency of the pulses is proportional to the flow rate, allowing precise volumetric measurement.

## Try This! (Challenges)
1. **Interactive Reset button**: Add a push button on GPIO 15 that clears the accumulated volume total back to 0.0 Liters.
2. **Leakage Safety Warning**: Sound a buzzer (GPIO 12) if a slow flow is detected continuously for more than 5 minutes (indicating a pipe leak or open tap).
3. **Solenoid Shutoff relay**: Add a relay on GPIO 13 to control an emergency shutoff valve if the total volume exceeds a daily limit (e.g. 50 Liters).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Flow rate is always 0.0 | Sensor pin floating | Ensure `INPUT_PULLUP` is declared to establish a logic level reference |
| Volume accumulates too fast or slow | Incorrect calibration factor | Verify the sensor model; YF-S201 uses a standard factor of 7.5, other models may vary |
| Microcontroller crashes under flow | Interrupt conflict | Keep the ISR function very short; do not call complex functions (like `Serial.print` or `delay`) inside the ISR |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [151 - ESP32 DC Motor PID Speed Controller](151-esp32-dc-motor-pid-speed-controller.md)
- [102 - ESP32 Water Level Control Station](../intermediate/102-esp32-water-level-control-station.md)
- [58 - ESP32 16×2 I2C LCD Print Text](../intermediate/58-esp32-16x2-i2c-lcd-print-text.md)
