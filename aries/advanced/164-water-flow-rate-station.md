# 164 - Water Flow Rate Station

Measure volumetric water flow rate in litres per minute by counting pulses from a YF-S201 hall-effect flow sensor connected to GPIO 17, and display the live flow rate and total volume dispensed on an I2C LCD.

## Goal
Learn how to count external digital pulses using state-machine-based edge detection (without interrupts, in interpreted mode), convert pulse frequency to flow rate, and accumulate total volume dispensed. Understand the YF-S201 sensor's frequency-to-flow relationship.

## What You Will Build
A YF-S201 flow sensor generates a series of digital pulses as water flows through it — approximately 7.5 pulses per second per litre per minute. GPIO 17 monitors the pulse output. The board detects each rising edge by comparing the current pin state to the previous state, counts pulses over a 1-second window, converts the count to flow rate (L/min), accumulates total litres, and displays both on a 16×2 I2C LCD and the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| YF-S201 Water Flow Sensor | `flow_sensor` | Yes | Yes |
| I2C LCD 16×2 (PCF8574 backpack) | `lcd_i2c` | Yes | Yes |
| 10 kΩ Pull-up Resistor | `resistor` | No | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| YF-S201 Flow Sensor | VCC (Red) | 5V | Red | Sensor supply voltage |
| YF-S201 Flow Sensor | GND (Black) | GND | Black | Common ground |
| YF-S201 Flow Sensor | Signal (Yellow) | GPIO 17 | Yellow | Pulse output — needs pull-up |
| Pull-up Resistor | Signal to VCC | — | — | 10 kΩ from Signal to 5V |
| I2C LCD | VCC | 5V | Red | LCD power |
| I2C LCD | GND | GND | Black | LCD ground |
| I2C LCD | SDA | SDA (GPIO 4) | Blue | I2C data |
| I2C LCD | SCL | SCL (GPIO 5) | Green | I2C clock |

> **Wiring tip:** The YF-S201 output is an open-collector signal that requires a pull-up resistor. A 10 kΩ resistor from the Signal wire to 5V is sufficient. The signal swings between 0 V and 5 V; use a voltage divider (1 kΩ + 2 kΩ) to step it down to 3.3 V if the ARIES board GPIOs are not 5 V tolerant. Check the ARIES v3 GPIO datasheet for the maximum input HIGH voltage before connecting directly.

## Code
```cpp
// Water Flow Rate Station
// YF-S201 pulse input: GPIO 17
// I2C LCD 16x2 at address 0x27

#include <LiquidCrystal_I2C.h>

#define FLOW_PIN  17

// YF-S201 calibration factor
// Sensor outputs ~7.5 pulses per second per L/min
const float PULSES_PER_LITRE = 450.0f;   // at typical flow (7.5 Hz per L/min * 60s)

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Pulse counting state
int  lastPinState  = LOW;
int  curPinState   = LOW;
long pulseCount    = 0L;
long pulseWindow   = 0L;   // Pulses counted in current 1-second window

// Timing state
long msTimer  = 0L;

// Flow measurements
float flowRate_Lmin = 0.0f;   // Litres per minute
float totalLitres   = 0.0f;   // Accumulated total volume

void setup() {
  Serial.begin(115200);
  pinMode(FLOW_PIN, INPUT_PULLUP);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Flow Station    ");
  lcd.setCursor(0, 1);
  lcd.print("Initialising... ");

  Serial.println("=== Water Flow Rate Station ===");
  Serial.println("Flow(L/min) | Total(L)");
  delay(1500);
}

void loop() {
  // Edge detection: count rising edges on FLOW_PIN
  curPinState = digitalRead(FLOW_PIN);
  if (curPinState == HIGH && lastPinState == LOW) {
    pulseWindow++;
    pulseCount++;
  }
  lastPinState = curPinState;

  // Accumulate time in a non-blocking manner (1 ms per loop assumed)
  msTimer++;
  delay(1);

  // Every 1000 ms: compute flow rate and update display
  if (msTimer >= 1000) {
    msTimer = 0;

    // Flow rate: pulses in last second -> L/min
    // YF-S201: frequency (Hz) / 7.5 = L/min
    flowRate_Lmin = (pulseWindow / 7.5f);

    // Accumulate total volume: L/min * (1s / 60s)
    totalLitres += flowRate_Lmin / 60.0f;

    pulseWindow = 0;

    // Update LCD
    lcd.setCursor(0, 0);
    lcd.print("Flow:");
    lcd.print(flowRate_Lmin, 2);
    lcd.print(" L/m  ");

    lcd.setCursor(0, 1);
    lcd.print("Total:");
    lcd.print(totalLitres, 3);
    lcd.print(" L  ");

    // Serial log
    Serial.print(flowRate_Lmin, 2);
    Serial.print(" L/min | ");
    Serial.print(totalLitres, 3);
    Serial.println(" L");
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Drag a **Digital Pulse Generator** (or **Button**) component and connect its output to **GPIO 17** to simulate flow sensor pulses.
3. Drag an **I2C LCD** component; connect **SDA** and **SCL** to the ARIES I2C pins.
4. Paste the code into the editor.
5. Select **Interpreted Mode** from the simulation dropdown.
6. Click **Run**.
7. Toggle the pulse generator or repeatedly click the button rapidly to simulate water flowing through the sensor; the LCD and Serial Monitor will show a rising flow rate and accumulated volume.

## Expected Output
Serial Monitor:
```
=== Water Flow Rate Station ===
Flow(L/min) | Total(L)
0.00 L/min | 0.000 L
3.47 L/min | 0.058 L
5.20 L/min | 0.145 L
5.47 L/min | 0.236 L
0.00 L/min | 0.236 L
```

LCD display:
```
Flow:5.20 L/m
Total:0.145 L
```

## Expected Canvas Behavior
* The LCD updates every second with the live flow rate and accumulated total volume.
* Each simulated pulse on GPIO 17 increments the internal pulse counter.
* When no pulses arrive for a full second, the flow rate drops to 0.00 L/min but total volume holds its last value.
* The Serial Monitor prints one row per second.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#define FLOW_PIN 17` | Declares GPIO 17 as the YF-S201 pulse input pin. |
| `PULSES_PER_LITRE = 450.0f` | Calibration constant based on 7.5 Hz per L/min × 60 seconds. |
| `pinMode(FLOW_PIN, INPUT_PULLUP)` | Enables the internal pull-up on GPIO 17 to keep the line HIGH when no pulses arrive. |
| `if (curPinState == HIGH && lastPinState == LOW)` | Rising-edge detection: increments counter only on the 0→1 transition. |
| `msTimer++; delay(1)` | Counts milliseconds in a non-blocking accumulator instead of a single long `delay`. |
| `flowRate_Lmin = (pulseWindow / 7.5f)` | Converts pulse frequency (pulses/second) to flow rate (L/min) using the YF-S201 factor. |
| `totalLitres += flowRate_Lmin / 60.0f` | Adds the volume delivered in the last second (L/min ÷ 60 = L/s × 1 s). |
| `pulseWindow = 0` | Resets the 1-second pulse window counter after computing flow rate. |

## Hardware & Safety Concept
* **Hall-Effect Flow Sensing**: The YF-S201 contains a spinning rotor with a small magnet and a fixed Hall-effect sensor. Each full rotor revolution generates one pulse. The rotor speed is proportional to flow velocity and therefore volumetric flow rate. The relationship is approximately linear between 1 and 30 L/min.
* **Pulse Counting Without Interrupts**: In MbedO interpreted mode, hardware interrupts (`attachInterrupt`) are not available. The code instead samples the GPIO pin every millisecond and detects rising edges by comparing consecutive states. This software polling approach works well up to a few hundred Hz (adequate for flow rates below ~20 L/min with the YF-S201).
* **Volume Accumulation Accuracy**: Each 1-second sample window introduces ±1 pulse counting error due to pulse timing alignment. For high-accuracy totalization, use a hardware interrupt counter in compiled mode. In interpreted mode, keep flow rates moderate to maintain acceptable accuracy.

## Try This! (Challenges)
1. **Flow Alert**: Add a warning LED on GPIO 15. Turn it on when the flow rate exceeds 10 L/min (potential pipe burst) and off when flow rate returns below 8 L/min (hysteresis), displaying an alarm message on the LCD.
2. **Resettable Total**: Read a push button on GPIO 16. When pressed, reset `totalLitres` to 0.0 and display "Volume Reset!" briefly on the LCD, then return to normal operation.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Flow rate always 0 | GPIO 17 not receiving pulses or pull-up missing | Verify FLOW_PIN wiring; confirm 10 kΩ pull-up is installed; check pulse generator is connected. |
| Flow rate very high or unstable | Electrical noise on the signal line | Add a 100 nF ceramic capacitor from the signal wire to GND to filter glitches. |
| LCD not updating | Wrong I2C address or SDA/SCL swapped | Try address 0x3F; verify SDA and SCL connections match ARIES I2C pins. |
| Total volume drifts when no flow | Noise pulses counted at rest | Add a noise filter: only count pulses after two consecutive HIGH readings to avoid single-spike errors. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [163 - Water Quality Station](163-water-quality-station.md)
- [165 - Smart Door Lock with PIR Auto-Relock](165-smart-door-lock.md)
- [166 - Smart Fan with Temperature Hysteresis](166-smart-fan.md)
