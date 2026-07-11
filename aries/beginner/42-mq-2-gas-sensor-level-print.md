# 42 - MQ-2 Gas Sensor Level Print (GPIO Analog pin)

Measure and print gas concentration levels using an MQ-2 combustible gas and smoke sensor on the VEGA ARIES v3 board.

## Goal
Understand the working mechanism of Metal Oxide Semiconductor (MOS) gas sensors, hook up an MQ-2 sensor requiring high-current heater power, and write code to map analog values to estimated Parts Per Million (PPM) concentrations.

## What You Will Build
An MQ-2 gas sensor is connected to analog input pin `ADC2` (`GP28`). The sensor measures ambient concentrations of LPG, smoke, butane, propane, and methane. The board reads the sensor's output voltage, maps it to an approximate gas concentration in PPM (200 to 10,000 PPM), and prints it to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MQ-2 Gas Sensor Module | `mq2` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MQ-2 Sensor | VCC | 5V | Red | Power connection (MQ-2 requires 5V for heating element) |
| MQ-2 Sensor | AO (Analog Out) | ADC2 (GP28) | Yellow | Analog concentration signal |
| MQ-2 Sensor | GND | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The MQ-2 sensor requires 5V power because of its internal heating element. Connecting it to 3.3V will result in incorrect or extremely sluggish readings. However, its Analog Output (AO) voltage is safe for GP28 (ADC2) because of the onboard voltage dividing on typical breakout boards.

## Code
```cpp
// MQ-2 Gas Sensor Level Print - VEGA ARIES v3
const int MQ2_PIN = GP28; // ADC2

void setup() {
  // Initialize serial communication at 115200 bps
  Serial.begin(115200);
}

void loop() {
  // Read the raw 12-bit ADC value from the gas sensor
  int rawValue = analogRead(MQ2_PIN);
  
  // Map raw ADC (0-4095) to approximate PPM range (200 - 10000 ppm)
  // 200 PPM is clean air baseline; 10000 PPM represents heavy gas concentrations
  int ppm = map(rawValue, 0, 4095, 200, 10000);
  
  // Print readings to the Serial Monitor
  Serial.print("Gas Sensor ADC: ");
  Serial.print(rawValue);
  Serial.print(" | Approx Gas Level: ");
  Serial.print(ppm);
  Serial.println(" PPM");
  
  // Check readings every 1 second
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and an **MQ-2 Gas Sensor** onto the canvas.
2. Wire the MQ-2 sensor: VCC to **5V**, AO to **GP28 (ADC2)**, and GND to **GND**.
3. Paste the code into the editor.
4. Click **Run**.
5. Adjust the gas concentration slider on the MQ-2 sensor widget and observe the Serial Monitor.

## Expected Output
Serial Monitor:
```
System Initialized.
Gas Sensor ADC: 150 | Approx Gas Level: 558 PPM
Gas Sensor ADC: 2048 | Approx Gas Level: 5100 PPM
```

## Expected Canvas Behavior
* Adjusting the gas concentration slider on the MQ-2 sensor increases the PPM reading displayed in the virtual terminal.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int MQ2_PIN = GP28;` | Specifies the analog pin GP28 (ADC2) connected to the MQ-2 sensor output. |
| `analogRead(MQ2_PIN)` | Reads the raw 12-bit voltage from the sensor. |
| `map(rawValue, 0, 4095, 200, 10000)` | Maps the 12-bit ADC value to an approximate PPM range from 200 PPM (normal clean air) to 10,000 PPM (dangerous gas leaks). |

## Hardware & Safety Concept: Metal Oxide Semiconductor Sensors and Preheating
* **Metal Oxide Semiconductor (MOS) Sensors**: MQ sensors use a tin dioxide ($SnO_2$) sensing layer. When heated, oxygen is adsorbed onto the semiconductor surface, blocking electron flow. When combustible gases are introduced, they react with the adsorbed oxygen, releasing electrons and decreasing the sensing layer's resistance, causing the output voltage to rise.
* **Preheating**: Physical MQ sensors require a preheat period (typically 24 to 48 hours for first-use burn-in, and 60 seconds before each run) to stabilize the heating element and burn off impurities. During this preheating time, the readings will spike before settling down.

## Try This! (Challenges)
1. **Toxic Gas Warning System**: Add the active buzzer on `GPIO 14`. Sound a warning alarm (beep) if the gas concentration exceeds 2000 PPM.
2. **Gas Leak Level LEDs**: Map three status levels (Safe, Warning, Critical) to the onboard RGB LED channels (Green, Blue, Red).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| PPM readings are stuck at extremely high values | Insufficient preheat or wrong power pin | Verify the module is connected to the 5V rail rather than the 3.3V rail. If using physical hardware, wait 60 seconds for the heating coil to warm up |
| Sensor is physically hot to the touch | Normal behavior (to a limit) | The internal nickel-chromium heating wire is designed to keep the sensor hot (approx 50-60°C) to facilitate the chemical reactions. However, if the sensor smells like burning plastic, disconnect power immediately |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [41 - Water level sensor level print](41-water-level-sensor-level-print.md) (Previous project)
- [43 - Flame sensor digital alarm](43-flame-sensor-digital-alarm.md) (Next project)
