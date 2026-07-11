# 191 - Indoor Air Quality Monitor

Create an industrial indoor air quality monitoring station. The system integrates MQ-2 (smoke/combustible gas) and MQ-7 (carbon monoxide) sensors with a DHT22 temperature and humidity sensor, displays environmental telemetry on an I2C SSD1306 OLED screen, and automatically activates ventilation fans via relay during threshold breaches.

## Goal
Learn how to poll multiple analog gas sensors alongside digital 1-wire sensors, initialize and format text coordinates on an OLED graphics layout, and drive high-power relays dynamically using a loop-free C++ state machine.

## What You Will Build
An automatic hazard-prevention monitor. The ARIES v3 board polls MQ-2, MQ-7, and DHT22 sensors. It outputs real-time readouts to the OLED display. If smoke exceeds 400 ppm, carbon monoxide exceeds 400 ppm, or temperature exceeds 40°C, the system triggers the ventilation relay (GPIO 2) to activate fans, sounds the alarm buzzer on GPIO 14, and flashes the Warning LED on GPIO 15.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MQ-2 Gas/Smoke Sensor | `mq2` | Yes | Yes |
| MQ-7 CO Gas Sensor | `mq7` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| SSD1306 OLED Display (I2C) | `oled` | Yes | Yes |
| 5V Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MQ-2 Sensor | VCC | 5V | Red | Sensor heater power |
| MQ-2 Sensor | GND | GND | Black | Ground reference |
| MQ-2 Sensor | AO (Analog Out) | ADC2 (GP28) | Yellow | Analog gas level |
| MQ-7 Sensor | VCC | 5V | Red | Sensor heater power |
| MQ-7 Sensor | GND | GND | Black | Ground reference |
| MQ-7 Sensor | AO (Analog Out) | ADC0 (GP26) | Orange | Analog CO level |
| DHT22 Sensor | VCC | 3V3 | Red | Power input |
| DHT22 Sensor | GND | GND | Black | Ground reference |
| DHT22 Sensor | DATA | GPIO 12 | Yellow | One-wire data bus |
| SSD1306 OLED | VCC | 3V3 | Red | Power input |
| SSD1306 OLED | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA | SDA0 (GP17) | Blue | I2C Data line |
| SSD1306 OLED | SCL | SCL0 (GP16) | Yellow | I2C Clock line |
| 5V Relay | IN | GPIO 2 | Orange | Active HIGH trigger |
| 5V Relay | VCC | 5V | Red | Relay coil power |
| 5V Relay | GND | GND | Black | Ground reference |
| Active Buzzer | + | GPIO 14 | Orange | Buzzer control |
| Active Buzzer | - | GND | Black | Ground reference |
| Warning LED | Anode | GPIO 15 | Orange | LED control |
| Warning LED | Cathode | GND (via 220 Ω) | Black | Current-limiting resistor |

> **Wiring tip:** Standard MQ gas sensors require a warm-up phase (often 24–48 hours in hardware for calibration) because they contain internal heating elements. Be careful as they can draw up to 150mA of current, so connect their VCC to the 5V line, not the 3.3V line.

## Code
```cpp
// 191 - Indoor Air Quality Monitor
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C
#define DHTPIN 12
#define DHTTYPE DHT22

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
DHT dht(DHTPIN, DHTTYPE);

const int MQ2_PIN = 28;     // ADC2
const int MQ7_PIN = 26;     // ADC0
const int RELAY_PIN = 2;    // Relay control
const int BUZZER_PIN = 14;
const int WARN_LED_PIN = 15;

const int MQ_THRESHOLD = 400; // Limit for MQ-2 and MQ-7 values
const float TEMP_LIMIT = 40.0;

int gasAlert = 0;
int toggleTick = 0;
int loopCounter = 0;

float temp = 0.0;
float hum = 0.0;
int mq2Val = 0;
int mq7Val = 0;

void setup() {
  Serial.begin(115200);
  Wire.begin();

  dht.begin();
  display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR);
  display.clearDisplay();
  display.display();

  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(WARN_LED_PIN, OUTPUT);

  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(WARN_LED_PIN, LOW);

  Serial.println("Indoor Air Quality Monitor Online.");
}

void loop() {
  // Read MQ sensors
  mq2Val = analogRead(MQ2_PIN);
  mq7Val = analogRead(MQ7_PIN);

  // Read DHT22
  float readTemp = dht.readTemperature();
  float readHum = dht.readHumidity();

  // Validate DHT readings
  if (!isnan(readTemp)) temp = readTemp;
  if (!isnan(readHum)) hum = readHum;

  // Determine alert status
  if (mq2Val > MQ_THRESHOLD || mq7Val > MQ_THRESHOLD || temp > TEMP_LIMIT) {
    gasAlert = 1;
  } else {
    gasAlert = 0;
  }

  // Actuator control state machine
  if (gasAlert == 1) {
    digitalWrite(RELAY_PIN, HIGH); // Turn ON ventilation fan
    
    toggleTick++;
    if (toggleTick % 2 == 0) {
      digitalWrite(BUZZER_PIN, HIGH);
      digitalWrite(WARN_LED_PIN, HIGH);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(WARN_LED_PIN, LOW);
    }
  } else {
    digitalWrite(RELAY_PIN, LOW);  // Turn OFF fan
    digitalWrite(BUZZER_PIN, LOW);
    digitalWrite(WARN_LED_PIN, LOW);
  }

  // Update OLED display (no clearing loops, just redraw static labels and vars)
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);

  // Header Line
  display.setTextSize(1);
  display.setCursor(0, 0);
  if (gasAlert == 1) {
    display.print("!!! HAZARD ALERT !!!");
  } else {
    display.print("AIR MONITOR - SECURE");
  }

  // Metrics
  display.setCursor(0, 16);
  display.print("Temp: ");
  display.print(temp, 1);
  display.print(" C");

  display.setCursor(0, 28);
  display.print("Hum:  ");
  display.print(hum, 1);
  display.print(" %");

  display.setCursor(0, 40);
  display.print("Smoke (MQ2): ");
  display.print(mq2Val);

  display.setCursor(0, 52);
  display.print("CO (MQ7):    ");
  display.print(mq7Val);

  display.display();

  // Serial logging every ~1000 ms (2 * 500 ms delay)
  loopCounter++;
  if (loopCounter >= 2) {
    loopCounter = 0;
    Serial.print("Temp: ");
    Serial.print(temp, 1);
    Serial.print("C | Hum: ");
    Serial.print(hum, 1);
    Serial.print("% | MQ-2: ");
    Serial.print(mq2Val);
    Serial.print(" | MQ-7: ");
    Serial.print(mq7Val);
    Serial.print(" | Ventilation: ");
    Serial.println(gasAlert ? "ON" : "OFF");
  }

  delay(500);
}
```

## What to Click in MbedO
1. Drag **VEGA ARIES v3**, **MQ-2**, **MQ-7**, **DHT22**, **SSD1306 OLED**, **Relay**, **Active Buzzer**, and **Warning LED** onto the canvas.
2. Wire the OLED: **VCC → 3V3**, **GND → GND**, **SDA → SDA0 (GP17)**, **SCL → SCL0 (GP16)**.
3. Wire the DHT22: **VCC → 3V3**, **GND → GND**, **DATA → GPIO 12**.
4. Wire MQ-2: **AO → ADC2 (GP28)**; MQ-7: **AO → ADC0 (GP26)**.
5. Wire Relay: **IN → GPIO 2**, **VCC → 5V**, **GND → GND**.
6. Wire Buzzer: **+ → GPIO 14**, **- → GND**; Warning LED: **Anode → GPIO 15**, **Cathode → GND** (via 220 Ω).
7. Paste the code, select **Interpreted Mode**, and click **Run**.
8. Move the sliders on MQ-2 or MQ-7 widget past 400, or temperature slider past 40°C. Observe the relay widget activation, buzzer chirps, and warning LED flashing.

## Expected Output
Serial Monitor:
```
Indoor Air Quality Monitor Online.
Temp: 24.5C | Hum: 45.0% | MQ-2: 120 | MQ-7: 85 | Ventilation: OFF
Temp: 24.5C | Hum: 45.2% | MQ-2: 121 | MQ-7: 88 | Ventilation: OFF
Temp: 24.5C | Hum: 45.0% | MQ-2: 480 | MQ-7: 92 | Ventilation: ON
Temp: 25.0C | Hum: 44.8% | MQ-2: 485 | MQ-7: 95 | Ventilation: ON
```

## Expected Canvas Behavior
* Normal state: OLED shows secure header, relay remains un-triggered (OFF), and indicators are quiet.
* Over-threshold state: The OLED displays `!!! HAZARD ALERT !!!`, the relay clicks ON, and both the Warning LED and buzzer flash/beep in unison.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `display.begin(...)` | Mounts the I2C OLED display controller to update the pixel canvas. |
| `dht.readTemperature()` | Reads the temperature value in Celsius over the 1-wire protocol. |
| `analogRead(MQ2_PIN)` | Captures the voltage rating from the MQ-2 sensor (0 to 1023). |
| `mq2Val > MQ_THRESHOLD` | Evaluates if the current air pollution metric crosses the safety mark. |
| `digitalWrite(RELAY_PIN, HIGH)` | Drives GPIO 2 high to energize the relay electromagnetic coil. |
| `display.display()` | Flushes the SSD1306 graphic buffer to the physical OLED screen. |

## Hardware & Safety Concept
* **MOS Gas Sensors:** Metal Oxide Semiconductor (MOS) sensors like the MQ-2 and MQ-7 utilize a tin dioxide ($SnO_2$) sensing layer. When heated, oxygen is adsorbed on the surface, creating a potential barrier. In the presence of reducing gases (CO, LPG, smoke), the oxygen surface density decreases, dropping the sensor resistance and increasing the analog voltage output.
* **Ventilation Relay Isolation:** Exhaust fans draw high AC currents. Using a mechanical or solid-state relay isolates the low-voltage ARIES board logic from the high-voltage mains power, protecting the microcontroller from induction currents.

## Try This! (Challenges)
1. **Dynamic Fan Speed Control:** Use a transistor circuit on GPIO 2 instead of a relay, and write a PWM signal scale (`analogWrite`) to adjust the fan speed proportionally to the gas level.
2. **CO Accumulation Tracker:** Create a state variable that records if MQ-7 values stay above 300 for longer than 30 consecutive seconds, triggering a latching warning even if levels drop temporarily.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Gas readings are constant at 1023 | Sensor heater not powered properly | Ensure VCC is connected to the 5V rail (heaters require ~150mA). |
| OLED shows corrupted characters | I2C address mismatch | Verify your SSD1306 address is `0x3C`. Clean connections to GP17 and GP16. |
| Ventilation relay jitters rapidly | Threshold border noise | Implement a hysteresis gap: turn fan ON at 400, but only turn it OFF when gas drops below 350. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [60 - OLED SSD1306 Display Setup (I2C)](../intermediate/60-oled-ssd1306-display-setup.md)
- [71 - DHT22 Temp & Humidity Serial Logs](../intermediate/71-dht22-temp-humidity-serial.md)
- [98 - Gas Leakage Alarm Node](../intermediate/98-gas-leakage-alarm-node.md)
