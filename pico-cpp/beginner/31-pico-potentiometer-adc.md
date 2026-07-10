# 31 - Pico Potentiometer ADC

Read analog voltage levels from a potentiometer using the Pico's Analog-to-Digital Converter (ADC).

## Goal
Learn how to read analog inputs, convert them to 12-bit digital values, and print voltage readings to the Serial Monitor.

## What You Will Build
An analog voltage meter:
- **Potentiometer (GP26)**: Connected to ADC0. Toggling the dial changes the raw ADC value printed to the Serial Monitor (from `0` to `4095`).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Potentiometer | Pin 1 (VCC) | 3.3V_OUT | Reference voltage |
| Potentiometer | Pin 2 (Wiper) | GP26 (ADC0)| Analog signal |
| Potentiometer | Pin 3 (GND) | GND | Ground return |

## Code
```cpp
const int POT_PIN = 26; // GP26 (ADC0)

void setup() {
  Serial.begin(9600);
  pinMode(POT_PIN, INPUT); // Configures pin for analog input
  Serial.println("ADC Monitor Active");
}

void loop() {
  int rawValue = analogRead(POT_PIN); // Reads 0 to 4095
  
  // Calculate voltage: rawValue * (3.3V / 4095)
  float voltage = rawValue * (3.3 / 4095.0);

  Serial.print("Raw Value: ");
  Serial.print(rawValue);
  Serial.print(" | Voltage: ");
  Serial.print(voltage);
  Serial.println(" V");

  delay(500); // Read twice per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **Potentiometer** onto the canvas.
2. Connect Potentiometer: **Pin 1** to **3V3**, **Pin 2** to **GP26**, **Pin 3** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the potentiometer slider on the canvas and look at the Serial logs.

## Expected Output

Terminal:
```
ADC Monitor Active
Raw Value: 2048 | Voltage: 1.65 V
Raw Value: 4095 | Voltage: 3.30 V
```

## Expected Canvas Behavior
| Potentiometer Slider Position | GP26 Read Value | Serial Voltage |
| --- | --- | --- |
| Leftmost (Min) | 0 | 0.00 V |
| Center | 2048 | 1.65 V |
| Rightmost (Max) | 4095 | 3.30 V |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `analogRead(POT_PIN)` | Initiates an ADC conversion and returns a 12-bit resolution value between `0` and `4095`. |
| `rawValue * (3.3 / 4095.0)` | Mapped scaling formula converting digital levels back to analog volts. |

## Hardware & Safety Concept: RP2040 ADC Architecture
The RP2040 chip contains a 12-bit SAR (Successive Approximation Register) ADC. It can sample up to 5 analog channels (4 external GP26-GP29, and 1 internal temperature sensor). The input voltage range is strictly bounded by 0V to 3.3V. Applying any voltage higher than 3.3V directly to an ADC pin will permanently damage the RP2040 input block.

## Try This! (Challenges)
1. **Low Voltage Alert**: Flash an external LED on GP15 if the voltage drops below 1.0V.
2. **Speed Dial**: Shorten the loop delay to 100 ms to log rapid dial adjustments.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Raw reading remains static at 4095 | Potentiometer GND disconnected | Check that the ground return wire is firmly tied to the Pico's GND pin. |

## Mode Notes
This basic analog project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [32 - Pico Potentiometer Dimmer](32-pico-potentiometer-dimmer.md)
- [46 - Pico Pot Servo](46-pico-pot-servo.md)
