# 138 - SD Card Current/Energy Logger

Measure current consumption using an ACS712 current sensor and log the raw ADC values and converted milliampere (mA) readings to a CSV file on an SD card using the VEGA ARIES v3 board.

## Goal
Learn how to interface analog sensors with the VEGA ARIES v3 analog-to-digital converter (ADC), perform calibration math to calculate current from raw voltage values, and log high-frequency electrical telemetry to an SD card.

## What You Will Build
An energy telemetry logger. Every 1 second, the board reads the analog signal from an ACS712 current sensor connected to ADC0 (GP26). It calculates the active current consumption in milliamperes (mA). The data is appended to `energy.csv` on the SD card in a three-column format: Timestamp (ms), Raw ADC Value, and Calculated Current (mA).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Micro SD Card Module | `sd_card` | Yes | Yes |
| ACS712 Current Sensor | `acs712` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SD Card Module | VCC | 5V | Red | Primary power |
| SD Card Module | GND | GND | Black | Ground reference |
| SD Card Module | CS | GP10 | Blue | SPI Slave Select |
| SD Card Module | MOSI | GP11 | Yellow | SPI Master Out |
| SD Card Module | MISO | GP12 | Green | SPI Master In |
| SD Card Module | SCK | GP13 | White | SPI Serial Clock |
| ACS712 Sensor | VCC | 5V | Orange | Power supply (5V) |
| ACS712 Sensor | OUT | GP26 | Purple | Analog output connected to ADC0 |
| ACS712 Sensor | GND | GND | Grey | Ground reference |

> **Wiring tip:** The ACS712 requires a stable 5V supply to establish its 2.5V zero-current reference. Connecting it to a 3.3V supply will result in highly distorted and invalid readings.

## Code
```cpp
#include <SPI.h>
#include <SD.h>

const int chipSelect = 10;
const int currentPin = GP26; // ADC0 pin
unsigned long lastLogTime = 0;

void setup() {
  Serial.begin(115200);
  
  // Wait for peripherals to settle
  delay(1000);
  
  pinMode(currentPin, INPUT);
  Serial.println("Initializing SD card...");

  if (!SD.begin(chipSelect)) {
    Serial.println("SD Card initialization failed!");
  } else {
    Serial.println("SD Card initialized successfully.");
  }
}

void loop() {
  unsigned long currentTime = millis();

  // Log electrical telemetry every 1000 ms (1 second)
  if (currentTime - lastLogTime >= 1000) {
    int sensorValue = analogRead(currentPin);

    // ACS712 Calibration:
    // Midpoint is 512 (2.5V reference for 0A current).
    // Sensitivity is 185mV per Amp for 5A model.
    // mA = (analogValue - 512) * (5V / 1023) / 0.185 * 1000
    // We approximate using integer math: mA = (sensorValue - 512) * 26
    int currentMA = (sensorValue - 512) * 26;
    if (currentMA < 0) {
      currentMA = 0; // Cut off negative offsets for DC monitoring
    }

    // Open file for appending
    File dataFile = SD.open("energy.csv", FILE_WRITE);

    if (dataFile) {
      dataFile.print(currentTime);
      dataFile.print(",");
      dataFile.print(sensorValue);
      dataFile.print(",");
      dataFile.println(currentMA);
      dataFile.close();

      // Print status to Serial
      Serial.print("Uptime: ");
      Serial.print(currentTime);
      Serial.print(" ms | ADC: ");
      Serial.print(sensorValue);
      Serial.print(" | Current: ");
      Serial.print(currentMA);
      Serial.println(" mA");
    } else {
      Serial.println("Error opening energy.csv");
    }

    lastLogTime = currentTime;
  }
  
  delay(10); // Debounce delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **SD Card Module**, and **ACS712 Current Sensor** onto the canvas.
2. Connect the SD Card module pins: **VCC** to **5V**, **GND** to **GND**, **CS** to **GP10**, **MOSI** to **GP11**, **MISO** to **GP12**, and **SCK** to **GP13**.
3. Connect the ACS712 pins: **VCC** to **5V**, **GND** to **GND**, and **OUT** to **GP26**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the program.
7. Click the ACS712 Current Sensor widget on the canvas to adjust the current levels, and verify the changes in the logs.

## Expected Output
Serial Monitor:
```
System Initialized.
Initializing SD card...
SD Card initialized successfully.
Uptime: 1000 ms | ADC: 512 | Current: 0 mA
Uptime: 2000 ms | ADC: 520 | Current: 208 mA
Uptime: 3000 ms | ADC: 532 | Current: 520 mA
```

## Expected Canvas Behavior
* The ACS712 widget dynamically represents current flows based on slider input.
* The SD Card Module logs the entries as comma-separated values, simulating physical file writes.
* Serial outputs report updated measurements once every second.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(currentPin, INPUT)` | Configures pin GP26 as an analog input to receive voltage inputs. |
| `analogRead(currentPin)` | Reads the voltage on GP26, converting it to a value between 0 (0V) and 1023 (5V). |
| `(sensorValue - 512) * 26` | Math logic to calculate mA from baseline deviations based on ACS712 sensitivity. |
| `SD.open("energy.csv", FILE_WRITE)` | Opens `energy.csv` in append mode. New logs are added to the end of the file. |
| `dataFile.close()` | Saves the data to the card, preventing corruption or block loss. |

## Hardware & Safety Concept
* **Hall Effect Sensitivity**: The ACS712 uses a Hall effect sensor to measure magnetic fields generated by current passing through the terminal pins. It is highly sensitive to stray magnetic fields (like those from nearby motors, transformers, or relays). Ensure physical sensor boards are kept away from electromagnetic noise sources.
* **Isolation**: The current sensing terminals are electrically isolated from the sensor's signal lines. This allows safe measurement of high-voltage loads without risk of damage to the low-voltage ARIES microcontroller.

## Try This! (Challenges)
1. **Accumulate Energy (Wh)**: Calculate power (W = Current * 5V) and integrate over time to estimate watt-hours (Wh) logged to the file.
2. **Overcurrent Flag**: Log a text warning `"OVERCURRENT"` next to the row if the current exceeds 500 mA.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Current reads 0 mA even under load | Sensor output not connected | Verify that the OUT pin is wired to GP26 (ADC0) and that the sensor has a shared GND. |
| Current values fluctuate heavily | Unstable power supply | Ensure the ACS712 is powered by the 5V line and not sharing a power rail with active inductive loads. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [137 - SD Card Temperature Logger](137-sd-card-temperature-logger.md) (Previous project)
- [139 - Multi-Sensor Environmental HUD](139-multi-sensor-environmental-hud.md) (Next project)
