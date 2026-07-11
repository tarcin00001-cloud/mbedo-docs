# 193 - Energy Harvesting Node

Design an environmental energy harvester logger. The system monitors solar panel output voltage, wind turbine pulse frequency (anemometer), and battery voltage using the ARIES onboard ADC channels, computes real-time power metrics, and saves the data to an SPI SD card telemetry log.

## Goal
Learn how to count fast-pulsing digital inputs (anemometer) through state tracking, convert analog inputs to scaled engineering units (Volts), initialize and write to files using the SPI SD card driver, and manage power logging cycles without using loops.

## What You Will Build
An off-grid remote telemetry logger. A solar panel simulator outputs voltage to ADC0 (GP26), while a battery charging circuit is monitored on ADC1 (GP27). A wind speed sensor pulses GPIO 16 for each rotation. The ARIES v3 board counts these pulses. Every 5 seconds, the board computes the average wind speed and solar/battery voltages, appends these readings to `energy.csv` on the SD card, and prints them to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Micro SD Card Module | `sd_card` | Yes | Yes |
| Wind Sensor (Pulse Anemometer) | `anemometer` (or button) | Yes | Yes |
| Solar Panel (0-18V output range) | `analog_sensor` | Yes | Yes |
| Rechargeable LiPo Battery (3.7V) | `battery` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SD Card Module | VCC | 5V | Red | 5V power supply |
| SD Card Module | GND | GND | Black | Ground reference |
| SD Card Module | CS | GPIO 10 | Blue | SPI Chip Select |
| SD Card Module | MOSI | GP11 (SPI MOSI) | Yellow | SPI Master Out Slave In |
| SD Card Module | MISO | GP12 (SPI MISO) | Green | SPI Master In Slave Out |
| SD Card Module | SCK | GP13 (SPI SCK) | White | SPI Serial Clock |
| Solar panel (via divider) | Analog Out | ADC0 (GP26) | Blue | Scaled solar voltage input |
| LiPo Battery (via divider) | Analog Out | ADC1 (GP27) | Green | Scaled battery voltage input |
| Wind Sensor | Pulse Out | GPIO 16 | Yellow | Pulse switch output |
| Wind Sensor | GND | GND | Black | Ground reference |
| Warning LED | Anode | GPIO 15 | Orange | Low battery indicator |
| Warning LED | Cathode | GND (via 220 Ω) | Black | Current-limiting resistor |

> **Wiring tip:** A 12V solar panel cannot be connected directly to ARIES pins (max 3.3V). You must use a voltage divider (e.g. 100 kΩ and 22 kΩ resistors) to scale the voltage down to a safe range. In software, multiply the reading by the division ratio to reconstruct the true voltage.

## Code
```cpp
// 193 - Energy Harvesting Node
#include <SPI.h>
#include <SD.h>

const int chipSelect = 10;
const int SOLAR_PIN = 26;    // ADC0
const int BATT_PIN = 27;     // ADC1
const int WIND_PIN = 16;     // Digital input for pulse counting
const int LOW_BATT_LED = 15; // Warning LED pin

// Voltage divider scale factors (adjust according to resistor values)
const float SOLAR_SCALE = 5.54;  // E.g., for 100k/22k divider
const float BATT_SCALE = 1.45;   // E.g., for 10k/22k divider

int lastWindState = HIGH;
int windPulses = 0;
int loopCounter = 0;
int sdStatus = 0; // 0 = Uninitialized, 1 = Ok, 2 = Failed

float solarVoltage = 0.0;
float batteryVoltage = 0.0;
float windSpeed = 0.0;

void setup() {
  Serial.begin(115200);
  
  pinMode(WIND_PIN, INPUT_PULLUP);
  pinMode(LOW_BATT_LED, OUTPUT);
  digitalWrite(LOW_BATT_LED, LOW);

  Serial.println("Initializing SD Card...");
  if (SD.begin(chipSelect)) {
    Serial.println("SD Card ready.");
    sdStatus = 1;

    // Write CSV header if file doesn't exist
    File dataFile = SD.open("energy.csv", FILE_WRITE);
    if (dataFile) {
      // Check if file is new (size 0) and print header
      if (dataFile.size() == 0) {
        dataFile.println("Solar_V,Battery_V,Wind_Pulses,Wind_Speed_kmh");
      }
      dataFile.close();
    }
  } else {
    Serial.println("SD Card initialization failed!");
    sdStatus = 2;
  }
}

void loop() {
  // Read wind sensor pulse transitions (detect falling edge)
  int currentWindState = digitalRead(WIND_PIN);
  if (currentWindState == LOW && lastWindState == HIGH) {
    windPulses++;
  }
  lastWindState = currentWindState;

  // Process data and write logs every 5 seconds (100 * 50 ms loop cycles)
  loopCounter++;
  if (loopCounter >= 100) {
    loopCounter = 0;

    // Read solar voltage
    int rawSolar = analogRead(SOLAR_PIN);
    solarVoltage = (rawSolar * (3.3 / 1023.0)) * SOLAR_SCALE;

    // Read battery voltage
    int rawBatt = analogRead(BATT_PIN);
    batteryVoltage = (rawBatt * (3.3 / 1023.0)) * BATT_SCALE;

    // Calculate wind speed (e.g. 1 pulse/sec = 2.4 km/h)
    // 5-second sampling window: frequency = pulses / 5.0
    windSpeed = (windPulses / 5.0) * 2.4;

    // Low battery warning (e.g. below 3.4V)
    if (batteryVoltage < 3.4) {
      digitalWrite(LOW_BATT_LED, HIGH);
    } else {
      digitalWrite(LOW_BATT_LED, LOW);
    }

    // Print to Serial
    Serial.print("Solar: ");
    Serial.print(solarVoltage, 2);
    Serial.print(" V | Battery: ");
    Serial.print(batteryVoltage, 2);
    Serial.print(" V | Wind Speed: ");
    Serial.print(windSpeed, 2);
    Serial.println(" km/h");

    // Log to SD card if active
    if (sdStatus == 1) {
      File dataFile = SD.open("energy.csv", FILE_WRITE);
      if (dataFile) {
        dataFile.print(solarVoltage, 2);
        dataFile.print(",");
        dataFile.print(batteryVoltage, 2);
        dataFile.print(",");
        dataFile.print(windPulses);
        dataFile.print(",");
        dataFile.println(windSpeed, 2);
        dataFile.close();
        Serial.println("Telemetry successfully logged to SD card.");
      } else {
        Serial.println("Error opening energy.csv for writing.");
      }
    }

    // Reset pulse counter for next sampling window
    windPulses = 0;
  }

  delay(50); // High-frequency polling loop
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **SD Card Module**, **anemometer** (or button), and two **analog source widgets** (for solar and battery simulation) onto the canvas.
2. Wire the SD Card module SPI pins as shown in the wiring table. CS goes to **GPIO 10**.
3. Wire the Solar input to **ADC0 (GP26)**, Battery input to **ADC1 (GP27)**.
4. Wire the wind sensor pulse pin to **GPIO 16**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.
8. Adjust the Solar and Battery analog source sliders on the widgets. Click the wind sensor button rapidly to simulate wind cup rotations. Watch the SD card logs stream to the Serial Monitor.

## Expected Output
Serial Monitor:
```
Initializing SD Card...
SD Card ready.
Solar: 12.45 V | Battery: 3.82 V | Wind Speed: 4.80 km/h
Telemetry successfully logged to SD card.
Solar: 14.12 V | Battery: 3.95 V | Wind Speed: 9.60 km/h
Telemetry successfully logged to SD card.
Solar: 6.20 V | Battery: 3.75 V | Wind Speed: 0.00 km/h
Telemetry successfully logged to SD card.
```

## Expected Canvas Behavior
* On startup, the SD Card Module widget lights up green.
* Every 5 seconds, the Serial Monitor updates with current voltage and wind speed values.
* If you drag the Battery slider below 3.4V, the Warning LED on GPIO 15 illuminates.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SD.begin(chipSelect)` | Establishes the SPI protocol on GP10–13 and mounts the SD FAT filesystem. |
| `SD.open("energy.csv", ...)` | Opens the CSV database file for appending newer data logs. |
| `analogRead(SOLAR_PIN)` | Reads the divided analog voltage of the simulated solar array. |
| `windPulses++` | Tracks falling transitions on GP16 to tally rotation count. |
| `windSpeed = ...` | Computes wind velocity using the counted pulses and frequency scale factor. |
| `dataFile.println(...)` | Appends a row of comma-separated energy values to the SD card. |
| `dataFile.close()` | Safely commits the cached sectors to flash and clears the buffer. |

## Hardware & Safety Concept
* **Anemometer Debouncing:** Mechanical wind sensors use reed switches that close once per rotation. These contacts can vibrate on closing (bouncing), registering as multiple phantom pulses. In hardware, a 100nF capacitor can be placed across the switch to filter noise, while software tracks edge intervals.
* **Undervoltage Lockout (UVLO):** Rechargeable lithium-polymer batteries get damaged permanently if discharged below 3.0V. The system monitors battery health, triggers a low battery alarm LED at 3.4V, and in commercial nodes enters a deep sleep power-down mode below 3.2V to preserve battery chemistry.

## Try This! (Challenges)
1. **Cumulative Wh Calculation:** Create a `float cumulativeWattHours` state variable. Estimate current by assuming load resistance (e.g., $Current = V / R$), multiply by voltage and time ($Power \times Time$), and append this running energy total to the SD log.
2. **File Rotation:** Check the size of `energy.csv` before writing. If it exceeds 10 KB, rename it to `energy2.csv` or start a new file automatically without using loops.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| "Card initialization failed!" | Loose SPI connections | Re-verify GP10 (CS), GP11 (MOSI), GP12 (MISO), and GP13 (SCK). |
| Wind speed is always zero | Missing edge transition detection | Verify the pulse switch is pulled up to 3.3V and connects to GND when pressed. |
| Solar voltage reading is inaccurate | Incorrect scale factor | Calibrate the `SOLAR_SCALE` multiplier by checking the physical voltage with a multimeter. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [136 - SD Card Data Logger](../advanced/136-sd-card-data-logger.md)
- [138 - SD Card Current & Energy Logger](../advanced/138-sd-card-current-energy-logger.md)
