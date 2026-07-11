# 159 - Automatic Battery Capacity Tester

Measure a battery's remaining capacity by monitoring its terminal voltage through a resistor voltage divider on ADC0/GP26, then switching a known discharge load on and off via a relay on GPIO 15 to calculate discharge time and estimated mAh capacity.

## Goal
Learn how to combine analog voltage sensing with relay-controlled load switching to perform a practical battery capacity test. The board continuously monitors battery voltage and calculates capacity (mAh) from the measured discharge time until the cut-off voltage is reached.

## What You Will Build
A voltage divider feeds the battery terminal voltage to ADC0 (GP26). A relay on GPIO 15 switches a known-resistance load resistor across the battery terminals. The board records elapsed discharge time in seconds, calculates the current drawn from the known load resistance, and prints real-time voltage and estimated capacity to the Serial Monitor. When the battery voltage drops below a configurable cut-off threshold, the relay opens and the final capacity result is displayed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Resistor Voltage Divider (R1 10 kΩ, R2 10 kΩ) | `resistor` | No | Yes |
| Relay Module (5 V, active-HIGH) | `relay` | Yes | Yes |
| Load Resistor (e.g. 10 Ω / 5 W) | `resistor` | No | Yes |
| Test Battery (AA / Li-ion etc.) | — | No | Yes |
| Flyback Diode (1N4007) | `diode` | No | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Voltage Divider | Junction (mid-point) | ADC0 / GP26 | Yellow | Divider taps battery voltage to 0–3.3 V range |
| Voltage Divider | R1 top | Battery + | Red | R1 = 10 kΩ from battery positive |
| Voltage Divider | R2 bottom | GND | Black | R2 = 10 kΩ to ground |
| Relay Module | VCC | 5V | Red | Relay coil power |
| Relay Module | GND | GND | Black | Common ground |
| Relay Module | IN | GPIO 15 | Green | Control signal from ARIES |
| Relay Module | COM | Battery + | Orange | Switched supply to load |
| Relay Module | NO | Load Resistor + | Blue | Normally-open contact connects load |
| Load Resistor | Other leg | GND | Black | Current return through known resistance |
| Flyback Diode | Across relay coil | — | — | Cathode to VCC, anode to IN |

> **Wiring tip:** The voltage divider scales the battery voltage so the ADC input never exceeds 3.3 V. For a 4.2 V Li-ion cell with equal 10 kΩ resistors, the mid-point will be ≈ 2.1 V — safely within the ADC range. For a 9 V battery, use R1 = 30 kΩ and R2 = 10 kΩ to keep the maximum below 3.3 V. Always include the flyback diode across the relay coil to protect the GPIO pin from inductive spikes.

## Code
```cpp
// Automatic Battery Capacity Tester
// ADC0/GP26 = voltage divider mid-point, GPIO 15 = relay control

#define RELAY_PIN     15
#define VBAT_PIN      26   // ADC0 / GP26

const float DIVIDER_RATIO  = 2.0f;
const float VREF           = 3.3f;
const float ADC_MAX        = 4095.0f;
const float LOAD_OHMS      = 10.0f;
const float CUTOFF_VOLTAGE = 3.0f;
const int   SAMPLE_MS      = 1000;

float  batteryVoltage    = 0.0f;
float  currentAmps       = 0.0f;
float  capacityMah       = 0.0f;
long   elapsedSeconds    = 0L;
bool   testRunning       = false;

void setup() {
  Serial.begin(115200);
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);

  Serial.println("=== Automatic Battery Capacity Tester ===");
  Serial.println("Measuring open-circuit voltage...");
  delay(500);

  int rawOC = analogRead(VBAT_PIN);
  float vocv = (rawOC / ADC_MAX) * VREF * DIVIDER_RATIO;
  Serial.print("Open-circuit voltage: ");
  Serial.print(vocv, 3);
  Serial.println(" V");

  if (vocv > CUTOFF_VOLTAGE) {
    Serial.println("Battery OK — starting discharge test.");
    digitalWrite(RELAY_PIN, HIGH);
    testRunning = true;
  } else {
    Serial.println("Battery below cut-off. Test aborted.");
  }
}

void loop() {
  if (!testRunning) {
    return;
  }

  delay(SAMPLE_MS);

  int raw = analogRead(VBAT_PIN);
  batteryVoltage = (raw / ADC_MAX) * VREF * DIVIDER_RATIO;
  currentAmps    = batteryVoltage / LOAD_OHMS;
  capacityMah   += (currentAmps * 1000.0f) * (SAMPLE_MS / 1000.0f / 3600.0f);
  elapsedSeconds++;

  Serial.print("t=");
  Serial.print(elapsedSeconds);
  Serial.print("s | Vbat=");
  Serial.print(batteryVoltage, 3);
  Serial.print("V | I=");
  Serial.print(currentAmps * 1000.0f, 1);
  Serial.print("mA | Cap=");
  Serial.print(capacityMah, 2);
  Serial.println(" mAh");

  if (batteryVoltage <= CUTOFF_VOLTAGE) {
    digitalWrite(RELAY_PIN, LOW);
    testRunning = false;
    Serial.println("--- Cut-off voltage reached ---");
    Serial.print("Final capacity: ");
    Serial.print(capacityMah, 1);
    Serial.println(" mAh");
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and a **Relay** component onto the canvas.
2. Add a **Potentiometer** to simulate the voltage divider mid-point; connect its wiper output to **ADC0 / GP26**.
3. Connect the **Relay IN** pin to **GPIO 15**, **VCC** to **5V**, and **GND** to **GND**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** from the simulation dropdown.
6. Click **Run**.
7. Turn the potentiometer slider upward (simulating a charged battery above 3.0 V); the test begins and logs appear in the Serial Monitor.
8. Slowly turn the slider down to simulate discharge; when the simulated voltage falls to or below 3.0 V the relay opens and the final capacity is printed.

## Expected Output
Serial Monitor:
```
=== Automatic Battery Capacity Tester ===
Measuring open-circuit voltage...
Open-circuit voltage: 3.987 V
Battery OK — starting discharge test.
t=1s | Vbat=3.985V | I=398.5mA | Cap=0.11 mAh
t=2s | Vbat=3.970V | I=397.0mA | Cap=0.22 mAh
t=3s | Vbat=3.955V | I=395.5mA | Cap=0.33 mAh
...
--- Cut-off voltage reached ---
Final capacity: 1842.30 mAh
```

## Expected Canvas Behavior
* The relay component toggles closed when the test begins and opens when cut-off voltage is reached.
* The Serial Monitor updates every second with live voltage, current draw, and accumulated capacity.
* Rotating the potentiometer widget downward simulates battery depletion and triggers the end-of-test condition.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#define RELAY_PIN 15` | Assigns GPIO 15 as the relay coil control output. |
| `#define VBAT_PIN 26` | Assigns GP26 (ADC0) as the analog battery voltage input. |
| `DIVIDER_RATIO = 2.0f` | Scaling factor that converts ADC mid-point voltage back to full battery voltage (1:1 divider). |
| `LOAD_OHMS = 10.0f` | Known load resistance; discharge current is computed using Ohm's Law: I = V ÷ R. |
| `CUTOFF_VOLTAGE = 3.0f` | Battery protection threshold; discharge halts when terminal voltage falls to this level. |
| `analogRead(VBAT_PIN)` | Reads a 12-bit ADC value (0–4095) from the voltage divider mid-point. |
| `batteryVoltage = (raw / ADC_MAX) * VREF * DIVIDER_RATIO` | Converts raw ADC counts to real battery terminal voltage. |
| `capacityMah += ...` | Integrates current over each 1-second interval to accumulate mAh (coulomb counting). |
| `digitalWrite(RELAY_PIN, LOW)` | Opens the relay to safely disconnect the load at end of test. |

## Hardware & Safety Concept
* **Coulomb Counting**: The tester estimates capacity by integrating current over time. Each second, the current (derived from battery voltage ÷ load resistance) is multiplied by the sample interval in hours and accumulated: Capacity (mAh) = Σ I(mA) × Δt(h). Accuracy improves when the load resistance is precisely known and thermally stable.
* **Voltage Divider Safety**: Never apply a battery voltage higher than 3.3 V directly to a GPIO or ADC pin on the ARIES v3. The voltage divider scales the input into the safe 0–3.3 V ADC window. Always verify the ratio before connecting a new battery chemistry.
* **Relay Flyback**: When the relay coil de-energises, a back-EMF spike is generated. The 1N4007 flyback diode clamps this spike, preventing damage to the GPIO output driver.

## Try This! (Challenges)
1. **Variable Cut-off Voltage**: Read a second potentiometer on ADC1/GP27 to set the cut-off voltage dynamically at runtime, allowing different battery chemistries (NiMH, Li-ion, LiPO) to be tested without modifying the code.
2. **Efficiency Logging**: Track and print the internal resistance estimate by measuring voltage sag from open-circuit to on-load and computing R_internal = (V_oc − V_load) ÷ I_load at the start of the test.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Voltage always reads 0 V | Voltage divider not connected or ADC pin mismatch | Confirm mid-point is wired to ADC0/GP26 and `VBAT_PIN = 26` in code. |
| Battery voltage reads too high | Divider ratio incorrect for battery chemistry | Recalculate R1 and R2 so the maximum voltage maps to ≤ 3.3 V; update `DIVIDER_RATIO`. |
| Relay never closes | GPIO 15 not configured as OUTPUT or relay polarity inverted | Check `pinMode(RELAY_PIN, OUTPUT)` exists in `setup()`; some relays are active-LOW — invert the `digitalWrite` calls. |
| Test stops immediately | Battery already discharged below cut-off | Charge the battery; verify `CUTOFF_VOLTAGE` matches the chemistry. |
| Capacity result far off | Load resistor value differs from `LOAD_OHMS` | Measure the actual resistance with a multimeter and update the constant. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [160 - Ultrasonic Wind Anemometer](160-ultrasonic-wind-anemometer.md)
- [167 - Current Overload Contactor Breaker](167-current-overload-contactor-breaker.md)
- [162 - Solar Tracker Efficiency Logger](162-solar-tracker-efficiency-logger.md)
