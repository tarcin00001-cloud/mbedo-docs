# 199 - Smart Beehive Monitor

Build a remote apiculture monitoring node. The system tracks beehive health by measuring internal temperature and relative humidity using a DHT22, monitoring hive sound levels (buzzing frequency/amplitude) via an analog sound sensor, and checking honey weight accumulation using an HX711 load cell, logging the collected agricultural parameters to an SPI SD card.

## Goal
Learn how to implement multi-sensor monitoring arrays, parse complex agricultural telemetry, manage power-safe log cycles to an SD card, and drive safety indicators based on health thresholds without using loops or custom helper functions.

## What You Will Build
A beehive health telemetry logger. The ARIES v3 board monitors hive parameters. A DHT22 on GPIO 2 reads internal temperature (optimal is 32°C–35°C). An analog sound sensor on ADC1 (GP27) tracks colony buzzing levels, while an HX711 on GPIO 3 & 7 reads hive weight. Every 10 seconds, these metrics are saved to `beehive.csv` on the SD card. If the hive temperature drops below 15°C, exceeds 38°C, or buzzing spikes above a threshold, the system sounds an alert buzzer (GPIO 14) and lights the warning LED (GPIO 15).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Micro SD Card Module | `sd_card` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| Analog Sound/Decibel Sensor | `sound_sensor` | Yes | Yes |
| HX711 Weight Sensor Module | `hx711` | Yes | Yes |
| Load Cell (0-50kg) | `load_cell` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SD Card Module | CS | GPIO 10 | Blue | SPI Chip Select |
| SD Card Module | MOSI | GP11 (SPI MOSI) | Yellow | SPI Master Out |
| SD Card Module | MISO | GP12 (SPI MISO) | Green | SPI Master In |
| SD Card Module | SCK | GP13 (SPI SCK) | White | SPI Serial Clock |
| SD Card Module | VCC | 5V | Red | 5V Power supply |
| SD Card Module | GND | GND | Black | Ground reference |
| DHT22 Sensor | DATA | GPIO 2 | Yellow | One-wire data pin (moved to GP2 to avoid GP12 clash) |
| DHT22 Sensor | VCC | 3V3 | Red | Power supply |
| DHT22 Sensor | GND | GND | Black | Ground reference |
| Sound Sensor | OUT | ADC1 (GP27) | Orange | Analog sound amplitude output |
| HX711 Module | DOUT | GPIO 3 | Blue | Load cell serial data |
| HX711 Module | SCK | GPIO 7 | Grey | Load cell serial clock |
| HX711 Module | VCC | 5V | Red | Power supply |
| HX711 Module | GND | GND | Black | Ground reference |
| Active Buzzer | + | GPIO 14 | Orange | Warning buzzer |
| Warning LED | Anode | GPIO 15 | Orange | Warning LED |

> **Wiring tip:** The DHT22 sensor is connected to **GPIO 2** to avoid pin conflicts with the SD Card SPI bus, which uses GP10 (CS), GP11 (MOSI), GP12 (MISO), and GP13 (SCK). The HX711 DOUT and SCK pins are wired to **GPIO 3** and **GPIO 7**.

## Code
```cpp
// 199 - Smart Beehive Monitor
#include <SPI.h>
#include <SD.h>
#include <DHT.h>
#include <HX711.h>

const int SD_CS = 10;
const int DHT_PIN = 2;
const int SOUND_PIN = 27;     // ADC1
const int HX711_DOUT = 3;
const int HX711_SCK = 7;
const int BUZZER_PIN = 14;
const int WARN_LED = 15;

#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);
HX711 scale;

float temp = 0.0;
float hum = 0.0;
int soundVal = 0;
float weightGrams = 0.0;

int sdActive = 0;
int loopCounter = 0;
int alertState = 0; // 0 = Normal, 1 = Environmental Alert, 2 = Audio/Predator Alert
int beepTick = 0;

void setup() {
  Serial.begin(115200);

  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(WARN_LED, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(WARN_LED, LOW);

  dht.begin();
  
  scale.begin(HX711_DOUT, HX711_SCK);
  scale.set_scale(420.0); // Calibration factor
  scale.tare();

  Serial.println("Initializing SD Card...");
  if (SD.begin(SD_CS)) {
    Serial.println("SD Card mounted successfully.");
    sdActive = 1;

    // Check if csv needs header
    File dataFile = SD.open("beehive.csv", FILE_WRITE);
    if (dataFile) {
      if (dataFile.size() == 0) {
        dataFile.println("Temp_C,Hum_Pct,Sound_Raw,Weight_g");
      }
      dataFile.close();
    }
  } else {
    Serial.println("SD Card initialization failed!");
    sdActive = 0;
  }

  Serial.println("Smart Beehive Monitor Online.");
}

void loop() {
  // Read DHT22
  float rTemp = dht.readTemperature();
  float rHum = dht.readHumidity();
  if (!isnan(rTemp)) temp = rTemp;
  if (!isnan(rHum)) hum = rHum;

  // Read sound sensor
  soundVal = analogRead(SOUND_PIN);

  // Read scale weight
  if (scale.is_ready()) {
    float rWeight = scale.get_units(1);
    if (rWeight < 0.0) rWeight = 0.0;
    weightGrams = rWeight;
  }

  // Evaluate Hive Health and Alert States
  // Hive temperature must stay around 32-35°C. Alert if too cold (< 15°C) or hot (> 38°C)
  if (temp < 15.0 || temp > 38.0) {
    alertState = 1; // Temperature warning
  } else if (soundVal > 600) {
    alertState = 2; // Sound/Intrusion warning (wasps/bears/swarming)
  } else {
    alertState = 0; // Normal status
  }

  // Alert Signaling
  if (alertState == 1) {
    digitalWrite(WARN_LED, HIGH);
    // Slow warning beep (every 1 second = 10 ticks * 100 ms)
    beepTick++;
    if (beepTick % 10 < 2) {
      digitalWrite(BUZZER_PIN, HIGH);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
    }
  } 
  else if (alertState == 2) {
    digitalWrite(WARN_LED, HIGH);
    // Rapid buzzer alarm
    beepTick++;
    if (beepTick % 2 == 0) {
      digitalWrite(BUZZER_PIN, HIGH);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
    }
  } 
  else {
    digitalWrite(WARN_LED, LOW);
    digitalWrite(BUZZER_PIN, LOW);
  }

  // Log to SD card and Serial every 10 seconds (100 * 100 ms)
  loopCounter++;
  if (loopCounter >= 100) {
    loopCounter = 0;

    Serial.print("Temp: ");
    Serial.print(temp, 1);
    Serial.print("C | Hum: ");
    Serial.print(hum, 1);
    Serial.print("% | Buzz: ");
    Serial.print(soundVal);
    Serial.print(" | Honey Weight: ");
    Serial.print(weightGrams, 1);
    Serial.print("g | Alert: ");
    if (alertState == 0) Serial.println("NONE");
    if (alertState == 1) Serial.println("ENV_TEMP");
    if (alertState == 2) Serial.println("PREDATOR/SWARM");

    // Write to SD card
    if (sdActive == 1) {
      File dataFile = SD.open("beehive.csv", FILE_WRITE);
      if (dataFile) {
        dataFile.print(temp, 1);
        dataFile.print(",");
        dataFile.print(hum, 1);
        dataFile.print(",");
        dataFile.print(soundVal);
        dataFile.print(",");
        dataFile.println(weightGrams, 1);
        dataFile.close();
        Serial.println("Beehive logs successfully committed to SD Card.");
      } else {
        Serial.println("Error opening beehive.csv for logging.");
      }
    }
  }

  delay(100); // 100 ms loop cycle
}
```

## What to Click in MbedO
1. Drag **VEGA ARIES v3**, **SD Card Module**, **DHT22**, **Sound Sensor**, **HX711 Weight Sensor**, **Active Buzzer**, and **Warning LED** onto the canvas.
2. Wire SD Card SPI pins. CS to **GPIO 10**.
3. Wire DHT22: **DATA → GPIO 2**, **VCC → 3V3**, **GND → GND**.
4. Wire Sound sensor: **OUT → ADC1 (GP27)**.
5. Wire HX711: **DOUT → GPIO 3**, **SCK → GPIO 7**, **VCC → 5V**, **GND → GND**.
6. Wire Buzzer: **+ → GPIO 14**; Warning LED: **Anode → GPIO 15**.
7. Paste code, select **Interpreted Mode**, and click **Run**.
8. Adjust the DHT22, Sound, or weight sliders. Watch the values update and log to the Serial Monitor.

## Expected Output
Serial Monitor:
```
Initializing SD Card...
SD Card mounted successfully.
Smart Beehive Monitor Online.
Temp: 34.2C | Hum: 60.2% | Buzz: 210 | Honey Weight: 1450.0g | Alert: NONE
Beehive logs successfully committed to SD Card.
Temp: 34.2C | Hum: 60.1% | Buzz: 680 | Honey Weight: 1450.0g | Alert: PREDATOR/SWARM
Beehive logs successfully committed to SD Card.
```

## Expected Canvas Behavior
* Normal state: Onboard Warning LED is OFF, buzzer is silent, and logs are saved every 10 seconds.
* Temperature breach: Warning LED lights up, and the buzzer beeps slowly.
* Sound spike: The Warning LED lights up, and the buzzer beeps rapidly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SD.begin(SD_CS)` | Mounts the SD Card module using GP10 (CS) over SPI0. |
| `dht.readTemperature()` | Reads internal hive temperature via GPIO 2 wire. |
| `scale.get_units(1)` | Samples the HX711 scale to determine hive weight. |
| `alertState = 1` | Sets the temperature error state to trigger alarms. |
| `beepTick % 10 < 2` | Creates a slow buzzer pulse pattern without delay loops. |
| `dataFile.close()` | Saves the file pointer, flushing data to SD storage sectors. |

## Hardware & Safety Concept
* **Acoustic Swarm Detection:** Bees use acoustic buzzing to communicate. A healthy hive maintains a steady "hum" between 100Hz and 250Hz. If the queen leaves (swarming) or the hive is attacked, the buzz level increases dramatically. Bandpass filters or FFT analysis in hardware can isolate specific frequencies.
* **Weighing Mechanical Isolation:** Hive weighing requires load cells rated up to 100 kg to support the entire wooden structure. The weight of the honey increases gradually over the summer. Filtering daily spikes (caused by bees flying in and out) allows tracking of true honey accumulation.

## Try This! (Challenges)
1. **Low Power Mode Logging:** Modify the logic to turn off the HX711 excitation power pin (using a MOSFET on GPIO 8) when not weighing to extend battery life.
2. **Honey Rate-of-Change Tracker:** Keep a state variable of the daily average weight. Print an alarm message if the hive weight drops suddenly by more than 2kg, which indicates swarming or honey theft.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| SD Card fails to mount | MISO pin conflict | Check that DHT22 is wired to GPIO 2, leaving GPIO 12 dedicated to the SPI MISO. |
| Weight values drift with temperature | Strain gauge thermal expansion | Implement software compensation. Subtract offsets proportional to temperature changes. |
| Noise readings spike when wind blows | Wind vibration on sound sensor | Place the sound sensor inside a protective wind shield or recess it inside the hive entrance. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [96 - Sound Level OLED Decibel Meter](../intermediate/96-sound-level-oled-decibel-meter.md)
- [109 - Analog Scale Reader](../intermediate/109-analog-scale-reader.md)
- [193 - Energy Harvesting Node](193-energy-harvesting-node.md)
