# 162 - Solar Tracker Efficiency Logger

Track the sun using LDRs, measure solar panel output power with an ACS712 current sensor and a voltage divider, and log timestamped power efficiency data to the Serial Monitor (simulating SD Card CSV output) for later analysis.

## Goal
Learn to combine light tracking, analog current measurement, and voltage sensing to build a comprehensive solar energy data logger. Understand how to compute power (P = V × I) and format data as CSV records for post-processing or file storage.

## What You Will Build
Four LDRs on ADC0–ADC3 provide differential light signals that guide the panel orientation (tracking logic). An ACS712 current sensor module (5 A version, connected to a separate analog input) measures output current. A voltage divider on another ADC channel measures panel output voltage. Every 5 seconds the board computes power (W), formats the data as a CSV line, and prints it to the Serial Monitor in a format ready to paste into a spreadsheet or write to an SD card file.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| LDR × 4 | `ldr` | Yes | Yes |
| 10 kΩ Pull-down Resistor × 4 | `resistor` | No | Yes |
| ACS712 5A Current Sensor Module | `acs712` | Yes | Yes |
| Voltage Divider (R1 10 kΩ, R2 10 kΩ) | `resistor` | No | Yes |
| Solar Panel (small, 5–6 V) | — | No | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR Top-Left | Wiper | ADC0 / GP26 | Yellow | Voltage divider — 10 kΩ pull-down to GND |
| LDR Top-Right | Wiper | ADC1 / GP27 | Orange | Voltage divider — 10 kΩ pull-down to GND |
| LDR Bottom-Left | Wiper | ADC2 / GP28 | Green | Voltage divider — 10 kΩ pull-down to GND |
| LDR Bottom-Right | Wiper | ADC3 / GP29 | Blue | Voltage divider — 10 kΩ pull-down to GND |
| ACS712 Module | VCC | 5V | Red | Module power supply |
| ACS712 Module | GND | GND | Black | Common ground |
| ACS712 Module | VIOUT | ADC1 / GP27 | Purple | Analog current output — shares channel in simulation |
| Voltage Divider | Mid-point | ADC0 / GP26 | White | Panel output voltage measurement |
| Solar Panel | + | ACS712 IP+ | Red | Solar positive through current sensor |
| Solar Panel | – | GND | Black | Solar negative |

> **Wiring tip:** In the simulation, use the potentiometer on ADC0/GP26 to represent the voltage divider output (panel voltage) and the potentiometer on ADC1/GP27 to represent the ACS712 output (current). In hardware, the ACS712 VIOUT sits at 2.5 V with no current (VCC/2 for a 5 V module). One ampere of current shifts the output by 185 mV (ACS712-5A). Derive current as: I = (V_out − 2.5) ÷ 0.185. If using a 3.3 V ACS712 module, the quiescent point shifts proportionally — consult the module datasheet.

## Code
```cpp
// Solar Tracker Efficiency Logger
// LDRs: ADC0(GP26)=TL, ADC1(GP27)=TR, ADC2(GP28)=BL, ADC3(GP29)=BR
// Voltage divider: ADC0/GP26  |  ACS712 current: ADC1/GP27

#define LDR_TL    26   // ADC0
#define LDR_TR    27   // ADC1
#define LDR_BL    28   // ADC2
#define LDR_BR    29   // ADC3

// In this dual-use simulation layout, GP26 = panel voltage, GP27 = ACS712
#define VPANEL_PIN  26
#define ISENSE_PIN  27

const float VREF      = 3.3f;
const float ADC_MAX   = 4095.0f;
const float V_DIVIDER = 2.0f;    // Voltage divider ratio
const float ACS_ZERO  = 2.5f;    // ACS712 quiescent output voltage (5V supply)
const float ACS_MV_A  = 0.185f;  // mV per ampere — ACS712-05B

// Logging interval (ms)
const int   LOG_MS   = 5000;

// Tracker state
const int   TOLERANCE = 50;
const int   STEP_DEG  = 1;
int angleAz  = 90;
int angleEl  = 90;

// Measurement state
int   tl = 0;
int   tr = 0;
int   bl = 0;
int   br = 0;
int   avgLeft  = 0;
int   avgRight = 0;
int   avgTop   = 0;
int   avgBot   = 0;
int   diffH    = 0;
int   diffV    = 0;
float panelV   = 0.0f;
float panelI   = 0.0f;
float panelW   = 0.0f;
long  logCount = 0L;
long  msAcc    = 0L;

void setup() {
  Serial.begin(115200);
  Serial.println("=== Solar Tracker Efficiency Logger ===");
  Serial.println("Timestamp_ms,Az_deg,El_deg,Voltage_V,Current_A,Power_W");
}

void loop() {
  // Read LDRs for tracking
  tl = analogRead(LDR_TL);
  tr = analogRead(LDR_TR);
  bl = analogRead(LDR_BL);
  br = analogRead(LDR_BR);

  avgLeft  = (tl + bl) / 2;
  avgRight = (tr + br) / 2;
  avgTop   = (tl + tr) / 2;
  avgBot   = (bl + br) / 2;
  diffH = avgLeft - avgRight;
  diffV = avgTop  - avgBot;

  if (diffH > TOLERANCE)       angleAz -= STEP_DEG;
  else if (diffH < -TOLERANCE) angleAz += STEP_DEG;
  if (diffV > TOLERANCE)       angleEl += STEP_DEG;
  else if (diffV < -TOLERANCE) angleEl -= STEP_DEG;

  if (angleAz < 0)   angleAz = 0;
  if (angleAz > 180) angleAz = 180;
  if (angleEl < 10)  angleEl = 10;
  if (angleEl > 170) angleEl = 170;

  // Accumulate time for CSV log interval
  delay(100);
  msAcc += 100;

  if (msAcc >= LOG_MS) {
    msAcc = 0;
    logCount++;

    // Read panel voltage and ACS712 current
    int rawV = analogRead(VPANEL_PIN);
    int rawI = analogRead(ISENSE_PIN);

    panelV = (rawV / ADC_MAX) * VREF * V_DIVIDER;
    float vAcs = (rawI / ADC_MAX) * VREF;
    panelI = (vAcs - ACS_ZERO) / ACS_MV_A;
    panelW = panelV * panelI;

    // CSV log line
    Serial.print(logCount * LOG_MS);
    Serial.print(",");
    Serial.print(angleAz);
    Serial.print(",");
    Serial.print(angleEl);
    Serial.print(",");
    Serial.print(panelV, 3);
    Serial.print(",");
    Serial.print(panelI, 3);
    Serial.print(",");
    Serial.println(panelW, 3);
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Add four **LDR** components and connect their outputs to **ADC0/GP26**, **ADC1/GP27**, **ADC2/GP28**, and **ADC3/GP29**.
3. Add two **Potentiometer** components — one on **ADC0/GP26** (simulating panel voltage) and one on **ADC1/GP27** (simulating ACS712 current output).
4. Paste the code into the editor.
5. Select **Interpreted Mode** from the simulation dropdown.
6. Click **Run**.
7. The Serial Monitor will print a CSV header followed by a new data row every 5 seconds.
8. Adjust the LDR sliders to simulate tracker movement; adjust the potentiometers to simulate different panel output levels.

## Expected Output
Serial Monitor:
```
=== Solar Tracker Efficiency Logger ===
Timestamp_ms,Az_deg,El_deg,Voltage_V,Current_A,Power_W
5000,89,90,5.128,1.247,6.394
10000,88,90,5.115,1.239,6.338
15000,87,91,5.109,1.235,6.310
20000,87,91,5.107,1.233,6.297
```

## Expected Canvas Behavior
* The Serial Monitor prints a CSV header once at startup.
* A new comma-separated data row appears every 5 seconds containing timestamp, servo angles, panel voltage, current, and power.
* LDR slider changes cause the azimuth and elevation angles in the CSV output to change.
* Potentiometer adjustments change the voltage, current, and power columns in subsequent rows.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Serial.println("Timestamp_ms,Az_deg,...")` | Prints the CSV header once during `setup()` for spreadsheet compatibility. |
| `analogRead(LDR_TL)` | Reads 12-bit ADC value from the top-left LDR voltage divider. |
| `diffH = avgLeft - avgRight` | Horizontal brightness error used to drive azimuth correction. |
| `msAcc += 100` | Accumulates 100 ms per loop cycle without a blocking `delay(5000)`. |
| `if (msAcc >= LOG_MS)` | Triggers a CSV log write every 5000 ms (5 seconds). |
| `panelV = (rawV / ADC_MAX) * VREF * V_DIVIDER` | Converts ADC count to real panel output voltage. |
| `vAcs = (rawI / ADC_MAX) * VREF` | Converts ADC count to the ACS712 output voltage. |
| `panelI = (vAcs - ACS_ZERO) / ACS_MV_A` | Converts ACS712 voltage to amperes: zero at 2.5 V, 185 mV/A sensitivity. |
| `panelW = panelV * panelI` | Computes instantaneous output power in watts. |

## Hardware & Safety Concept
* **ACS712 Hall-Effect Current Sensing**: The ACS712 measures current by detecting the magnetic field produced by the primary current path through an integrated Hall-effect sensor. The output voltage is proportional to current: V_out = 2.5 + (I × 0.185) for a 5 A module on a 5 V supply. It provides galvanic isolation between the high-current panel circuit and the low-voltage ADC input, making it safe for the MCU.
* **CSV Logging Strategy**: Printing CSV directly to the Serial Monitor allows immediate data capture using any terminal emulator's log-to-file function. In hardware, replace the `Serial.println(...)` log lines with `SD.open("log.csv").println(...)` to write directly to an SD card.
* **Non-Blocking Interval Timing**: Instead of `delay(5000)` which would freeze the tracker every 5 seconds, the code uses `msAcc` to accumulate time across 100 ms tracking ticks, keeping the tracker responsive while still producing timed CSV rows.

## Try This! (Challenges)
1. **Efficiency Ratio**: Compute and log an efficiency percentage by dividing measured output power by an estimated maximum expected power (e.g., 5 V × 1 A = 5 W) and multiply by 100. This gives a quick indicator of tracking accuracy and cloud cover impact.
2. **Peak Power Tracker**: Record the maximum power seen since startup in a `float peakW` variable and add it as an additional CSV column, allowing peak performance periods to be identified in post-analysis.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Current reads negative | ACS712 VIOUT below 2.5 V, or reversed current direction | Verify the ADC reads above mid-scale (≈ 2048) at zero current; swap the panel leads if negative. |
| Power reads 0 W | Panel voltage or current reading 0 | Confirm potentiometer or sensor wiring on ADC0 and ADC1; check `VPANEL_PIN` and `ISENSE_PIN` constants. |
| CSV rows appear too frequently | `LOG_MS` too small or `msAcc` incrementing incorrectly | Verify `delay(100)` and `msAcc += 100` are both present in the loop body. |
| Tracker angles not changing | LDR differential below `TOLERANCE` | Adjust LDR sliders to create a larger imbalance; lower `TOLERANCE` if needed. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [161 - Solar Tracker Dual Axis Controller](161-solar-tracker-dual-axis.md)
- [167 - Current Overload Contactor Breaker](167-current-overload-contactor-breaker.md)
- [159 - Automatic Battery Capacity Tester](159-automatic-battery-capacity-tester.md)
