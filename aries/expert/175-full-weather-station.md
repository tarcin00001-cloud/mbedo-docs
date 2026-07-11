# 175 - Full Weather Station

Build a comprehensive weather logging station incorporating temperature, humidity, pressure, and wind speed tracking, with live readings displayed on an I2C OLED and logged to an SD card.

## Goal
Learn how to coordinate multiple sensors using diverse interfaces (I2C, SPI, One-Wire, Digital pulse counting), format live sensor values onto an OLED display, and log continuous records to an SD card without code blocking or loops.

## What You Will Build
An environmental monitoring station. A DHT22 reads temperature and humidity, a BMP180 measures barometric pressure, and a digital input pin measures pulse signals from an anemometer to calculate wind speed. Every 2 seconds, this telemetry data is logged to a CSV file on an SD card and displayed on a 128x64 OLED screen.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| BMP180 Pressure Sensor | `bmp180` | Yes | Yes |
| Anemometer (Pulse Output) | `pulse_generator` | Yes | Yes |
| I2C OLED Display (128x64)| `oled_i2c` | Yes | Yes |
| SD Card Reader Module | `sd_spiv3` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO 12 | Yellow | One-Wire communication |
| DHT22 Sensor | VCC | 3V3 | Red | Power |
| DHT22 Sensor | GND | GND | Black | Ground |
| BMP180 Sensor | SDA | GPIO 17 | Green | Shared I2C Data (SDA0) |
| BMP180 Sensor | SCL | GPIO 16 | Yellow | Shared I2C Clock (SCL0) |
| BMP180 Sensor | VCC | 3V3 | Red | Power |
| BMP180 Sensor | GND | GND | Black | Ground |
| OLED Display | SDA | GPIO 17 | Green | Shared I2C Data (SDA0) |
| OLED Display | SCL | GPIO 16 | Yellow | Shared I2C Clock (SCL0) |
| OLED Display | VCC | 5V / 3V3 | Red | Power |
| OLED Display | GND | GND | Black | Ground |
| SD Card Reader | CS | GPIO 8 | Purple | SPI Chip Select |
| SD Card Reader | SCK | GPIO 5 | Blue | SPI SCK |
| SD Card Reader | MOSI | GPIO 14 | Yellow | SPI COPI |
| SD Card Reader | MISO | GPIO 15 | Green | SPI CIPO |
| SD Card Reader | VCC | 5V | Red | Power |
| SD Card Reader | GND | GND | Black | Ground |
| Anemometer | OUT | GPIO 11 | Orange | Pulse signal count |
| Anemometer | VCC | 5V | Red | Power |
| Anemometer | GND | GND | Black | Ground |
| Active Buzzer | + | GPIO 14 | Gray | Storm alert alarm |
| Active Buzzer | - | GND | Black | Ground |
| Warning LED | Anode | GPIO 15 | Red | Status indicator |
| Warning LED | Cathode | GND | Black | Ground |

> **Wiring tip:** Standard I2C devices (OLED, BMP180) can share the physical I2C bus (SDA0, SCL0) because they have different bus addresses (OLED is usually 0x3C, BMP180 is 0x77). No external address multiplexer is needed.

## Code
```cpp
// Full Weather Station - VEGA ARIES v3
#include <SPI.h>
#include <SD.h>
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_BMP085.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int SD_CS = 8;
const int DHT_PIN = 12;
const int ANEMO_PIN = 11;
const int BUZZER = 14;
const int LED_PIN = 15;

DHT dht(DHT_PIN, DHT22);
Adafruit_BMP085 bmp;
Adafruit_SSD1306 display(128, 64, &Wire, -1);

// Anemometer variables
int lastAnemoState = LOW;
long windPulses = 0;
float windSpeed = 0.0; // in m/s

// Periodic log variables
unsigned long lastLogTime = 0;
unsigned long lastSampleTime = 0;

void setup() {
  Serial.begin(115200);
  Wire.begin();
  
  pinMode(ANEMO_PIN, INPUT);
  pinMode(BUZZER, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  
  dht.begin();
  
  if (!bmp.begin()) {
    Serial.println("Could not find BMP180 sensor!");
  }
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("SSD1306 allocation failed");
  }
  
  SD.begin(SD_CS);
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(WHITE);
  display.setCursor(0, 0);
  display.println("Weather Station");
  display.println("Initializing...");
  display.display();
  
  lastAnemoState = digitalRead(ANEMO_PIN);
  lastLogTime = millis();
}

void loop() {
  // Read anemometer pulses (state changes) continuously
  int currentAnemoState = digitalRead(ANEMO_PIN);
  if (currentAnemoState != lastAnemoState) {
    if (currentAnemoState == HIGH) {
      windPulses = windPulses + 1;
    }
    lastAnemoState = currentAnemoState;
  }
  
  unsigned long currentTime = millis();
  unsigned long elapsed = currentTime - lastLogTime;
  
  // Update log, OLED, and SD every 2 seconds
  if (elapsed >= 2000) {
    lastLogTime = currentTime;
    
    // Read DHT22
    float temp = dht.readTemperature();
    float hum = dht.readHumidity();
    
    // Read BMP180
    float pressure = bmp.readPressure(); // in Pa
    
    // Wind Speed calculation: 1 pulse per rotation, radius 0.05m
    // Circumference = 2 * PI * r = 0.314 meters
    // Wind Speed (m/s) = (Pulses / Time) * Circumference
    windSpeed = ((float)windPulses / (float)elapsed * 1000.0) * 0.314;
    windPulses = 0; // reset counter
    
    // Check for storm condition (high wind speed OR pressure drop)
    bool stormAlert = false;
    if (windSpeed > 10.0 || pressure < 99000.0) {
      stormAlert = true;
      digitalWrite(BUZZER, HIGH);
      digitalWrite(LED_PIN, HIGH);
    } else {
      digitalWrite(BUZZER, LOW);
      digitalWrite(LED_PIN, LOW);
    }
    
    // Log to SD
    File dataFile = SD.open("weather.csv", FILE_WRITE);
    if (dataFile) {
      dataFile.print(currentTime / 1000.0);
      dataFile.print(",");
      dataFile.print(temp);
      dataFile.print(",");
      dataFile.print(hum);
      dataFile.print(",");
      dataFile.print(pressure);
      dataFile.print(",");
      dataFile.print(windSpeed);
      dataFile.println(stormAlert ? ",ALERT" : ",NORMAL");
      dataFile.close();
    }
    
    // Display on OLED
    display.clearDisplay();
    display.setCursor(0, 0);
    display.print("Temp: ");
    display.print(temp, 1);
    display.println(" C");
    
    display.print("Humidity: ");
    display.print(hum, 1);
    display.println(" %");
    
    display.print("Pres: ");
    display.print(pressure / 100.0, 1);
    display.println(" hPa");
    
    display.print("Wind: ");
    display.print(windSpeed, 2);
    display.println(" m/s");
    
    if (stormAlert) {
      display.println("** STORM WARNING **");
    } else {
      display.println("System Normal");
    }
    display.display();
    
    // Debug output to Serial
    Serial.print("T:");
    Serial.print(temp);
    Serial.print(" H:");
    Serial.print(hum);
    Serial.print(" P:");
    Serial.print(pressure);
    Serial.print(" W:");
    Serial.println(windSpeed);
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **DHT22**, **BMP180**, **Pulse Generator** (simulating the anemometer), **I2C OLED (128x64)**, **SD Card Reader**, **Buzzer**, and **LED** onto the canvas.
2. Complete the connections as detailed in the wiring table.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the dropdown.
5. Click **Run**.
6. Modify the pulse frequency or DHT/BMP values on the sensor widgets and watch the OLED screen update.

## Expected Output
Serial Monitor:
```
T:24.50 H:45.00 P:101325.00 W:1.24
T:24.60 H:45.20 P:101310.00 W:2.82
T:23.90 H:55.80 P:98950.00 W:12.30 (STORM ALERT)
```

## Expected Canvas Behavior
* The I2C OLED display prints temperature, humidity, pressure, and wind speed.
* When the pulse generator speed widget is turned up high (simulating strong wind) or the pressure slider drops, the buzzer triggers and OLED prints "** STORM WARNING **".
* The SD card file `weather.csv` increments with logs every 2 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `display.begin(...)` | Sets up the SSD1306 controller over I2C0 using address 0x3C. |
| `digitalRead(ANEMO_PIN)` | High-frequency polling tracks pulses without locking loop execution. |
| `elapsed >= 2000` | Limits reads, log operations, and display refreshes to a 2-second interval. |
| `windSpeed = ...` | Translates raw state counts over the elapsed period into velocity based on blade sweep. |
| `SD.open("weather.csv", ...)`| Opens/creates a CSV file to save comma-delimited logs sequentially. |

## Hardware & Safety Concept
* **Shared I2C Bus Pull-Ups**: When running several high-speed I2C devices, trace capacitance can distort signal edges. Ensure the bus has appropriate pull-up resistors (typically 4.7 kΩ to 3.3V).
* **Power Quality for Anemometers**: Magnetic reed-switch based anemometers generate contact bounce. Standard debouncing capacitors (0.1 uF) should be added in parallel with the sensor output to clean up noisy trigger transitions.

## Try This! (Challenges)
1. **Average Wind Calculator**: Keep a running average of wind speeds over the last 10 samples without declaring arrays by tracking sum and count values.
2. **Altitude Calculator**: Use `bmp.readAltitude(101325)` to display altitude on the OLED, and light the warning LED when climbing above a target altitude.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED remains blank | Wrong I2C address | Change I2C address parameter from 0x3C to 0x3D in the `display.begin()` call. |
| Wind speed calculation stays at 0 | Sensor pin not registering pulses | Verify signal pin is connected to GPIO 11 and pulses are transitioning. |
| SD Card fails during log operation | SPI conflicts or voltage drop | Ensure SD CS is properly routed to GPIO 8. Verify the SD card is getting 5V power. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [74 - DHT22 Temperature OLED Graph](../intermediate/74-dht22-temperature-oled-graph.md)
- [137 - SD Card Temperature Logger](../advanced/137-sd-card-temperature-logger.md)
- [160 - Ultrasonic Wind Anemometer](../advanced/160-ultrasonic-wind-anemometer.md)
