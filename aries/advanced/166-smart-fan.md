# 166 - Smart Fan with Temperature Hysteresis

Control a relay-switched fan using a DHT22 temperature sensor on GPIO 12, applying separate ON and OFF temperature thresholds (hysteresis) to prevent the relay from rapidly toggling at the set-point boundary.

## Goal
Learn how to implement hysteresis control — an industrial technique that uses two separate thresholds to create a stable dead-band around the target set-point — using a DHT22 sensor and a relay output. Understand why hysteresis is essential for relay longevity and system stability.

## What You Will Build
A DHT22 sensor on GPIO 12 reads ambient temperature every 2 seconds. A relay on GPIO 15 switches the fan on and off. The fan turns ON when temperature rises above `TEMP_ON` (28 °C) and turns OFF when it falls below `TEMP_OFF` (26 °C). The 2 °C dead-band between the thresholds prevents relay chatter near the set-point. The onboard green LED (GPIO 24) indicates the fan is running; the red LED (GPIO 23) indicates standby. The Serial Monitor logs temperature, humidity, and fan state.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Temperature & Humidity Sensor | `dht22` | Yes | Yes |
| Relay Module (5 V, active-HIGH) | `relay` | Yes | Yes |
| DC Fan (5 V or 12 V) | — | No | Yes |
| Flyback Diode (1N4007) | `diode` | No | Yes |
| 10 kΩ Pull-up Resistor (DHT22) | `resistor` | No | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | VCC (Pin 1) | 3V3 | Red | Sensor power |
| DHT22 Sensor | DATA (Pin 2) | GPIO 12 | Yellow | One-wire data line |
| DHT22 Sensor | GND (Pin 4) | GND | Black | Ground reference |
| Pull-up Resistor | VCC to DATA | — | — | 10 kΩ from 3V3 to GPIO 12 |
| Relay Module | VCC | 5V | Red | Relay coil supply |
| Relay Module | GND | GND | Black | Common ground |
| Relay Module | IN | GPIO 15 | Orange | Control signal |
| Relay Module | COM | Fan supply + | Brown | Switched power |
| Relay Module | NO | Fan + | Blue | Normally-open fan connection |
| Fan | GND | GND | Black | Fan negative |
| Flyback Diode | Across relay coil | — | — | 1N4007 cathode to VCC, anode to GND |

> **Wiring tip:** Choose the relay NO (normally-open) contact so the fan is off by default when power is first applied. This is the safe fail state — if the ARIES board loses power or reboots, the fan will not run unattended. The flyback diode is mandatory to protect the GPIO driver from the relay coil's inductive kick when it de-energises.

## Code
```cpp
// Smart Fan with Temperature Hysteresis
// DHT22: GPIO 12  |  Relay: GPIO 15
// Fan ON above TEMP_ON, Fan OFF below TEMP_OFF

#include <DHT.h>

#define DHTPIN     12
#define DHTTYPE    DHT22
#define RELAY_PIN  15

// Hysteresis thresholds (degrees Celsius)
const float TEMP_ON  = 28.0f;  // Fan turns ON  above this
const float TEMP_OFF = 26.0f;  // Fan turns OFF below this

DHT dht(DHTPIN, DHTTYPE);

bool   fanOn      = false;
float  tempC      = 0.0f;
float  humidity   = 0.0f;

void setup() {
  Serial.begin(115200);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(LED_R, OUTPUT);
  pinMode(LED_G, OUTPUT);

  digitalWrite(RELAY_PIN, LOW);   // Fan off at startup
  digitalWrite(LED_R, HIGH);      // Red = standby
  digitalWrite(LED_G, LOW);

  dht.begin();

  Serial.println("=== Smart Fan with Temperature Hysteresis ===");
  Serial.print("Fan ON threshold:  ");
  Serial.print(TEMP_ON, 1);
  Serial.println(" C");
  Serial.print("Fan OFF threshold: ");
  Serial.print(TEMP_OFF, 1);
  Serial.println(" C");
  Serial.println("Temp(C) | Humidity(%) | Fan");
}

void loop() {
  delay(2000);

  tempC    = dht.readTemperature();
  humidity = dht.readHumidity();

  if (isnan(tempC) || isnan(humidity)) {
    Serial.println("DHT22 read error!");
    return;
  }

  // Hysteresis control logic
  if (!fanOn && tempC >= TEMP_ON) {
    // Temperature rose above ON threshold — turn fan on
    fanOn = true;
    digitalWrite(RELAY_PIN, HIGH);
    digitalWrite(LED_G, HIGH);
    digitalWrite(LED_R, LOW);
    Serial.println("[FAN ON]");
  } else if (fanOn && tempC <= TEMP_OFF) {
    // Temperature fell below OFF threshold — turn fan off
    fanOn = false;
    digitalWrite(RELAY_PIN, LOW);
    digitalWrite(LED_G, LOW);
    digitalWrite(LED_R, HIGH);
    Serial.println("[FAN OFF]");
  }

  // Log current readings
  Serial.print(tempC, 1);
  Serial.print(" C | ");
  Serial.print(humidity, 1);
  Serial.print("% | Fan=");
  Serial.println(fanOn ? "ON" : "OFF");
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Drag a **DHT22** component; connect **DATA** to **GPIO 12**, **VCC** to **3V3**, **GND** to **GND**.
3. Drag a **Relay** component; connect **IN** to **GPIO 15**, **VCC** to **5V**, **GND** to **GND**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** from the simulation dropdown.
6. Click **Run**.
7. Drag the DHT22 temperature slider above 28 °C — the relay should close and the green LED should light.
8. Drag the slider back down below 26 °C — the relay should open and the red LED should light.
9. Observe that the relay does NOT switch at exactly 27 °C — this is the hysteresis dead-band working correctly.

## Expected Output
Serial Monitor:
```
=== Smart Fan with Temperature Hysteresis ===
Fan ON threshold:  28.0 C
Fan OFF threshold: 26.0 C
Temp(C) | Humidity(%) | Fan
25.3 C | 55.0% | Fan=OFF
25.8 C | 55.2% | Fan=OFF
28.1 C | 56.0% | Fan=OFF
[FAN ON]
28.1 C | 56.0% | Fan=ON
27.5 C | 55.8% | Fan=ON
26.0 C | 55.5% | Fan=ON
25.9 C | 55.3% | Fan=ON
[FAN OFF]
25.9 C | 55.3% | Fan=OFF
```

## Expected Canvas Behavior
* The relay widget switches from open to closed when the temperature slider exceeds 28 °C.
* The green LED activates and the red LED deactivates simultaneously.
* Moving the slider to exactly 27 °C (between the two thresholds) holds the relay in its current state — no switching occurs.
* The Serial Monitor prints a `[FAN ON]` or `[FAN OFF]` event line only when the state actually changes.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <DHT.h>` | Imports the Adafruit DHT Sensor Library for communicating with the DHT22. |
| `TEMP_ON = 28.0f` | Upper threshold: fan activates when temperature rises above this value. |
| `TEMP_OFF = 26.0f` | Lower threshold: fan deactivates when temperature falls below this value. |
| `bool fanOn = false` | State variable tracking whether the fan is currently running. |
| `dht.readTemperature()` | Reads the current temperature in Celsius from the DHT22. |
| `isnan(tempC)` | Validates the sensor reading; skips the control update if the sensor fails. |
| `if (!fanOn && tempC >= TEMP_ON)` | ON condition: only activates if fan is currently off AND temperature exceeds the high threshold. |
| `else if (fanOn && tempC <= TEMP_OFF)` | OFF condition: only deactivates if fan is currently on AND temperature drops below the low threshold. |
| `digitalWrite(RELAY_PIN, HIGH)` | Energises the relay coil, closing the NO contact and connecting the fan to its supply. |

## Hardware & Safety Concept
* **Hysteresis Control**: Simple on/off (bang-bang) control without hysteresis causes rapid relay toggling whenever the temperature sits exactly at the set-point. This "relay chatter" generates heat in the relay contacts, produces electromagnetic interference, and dramatically shortens relay life (typically rated for 100,000 mechanical operations). Adding a dead-band between separate ON and OFF thresholds prevents state changes until a meaningful temperature excursion occurs, protecting both the relay and the controlled equipment.
* **Relay Contact Ratings**: Check that the relay's contact voltage and current ratings exceed the fan's requirements. A 5 V, 500 mA fan needs a relay rated for at least 1 A at 5 V DC. For AC fan loads, use a relay rated for AC voltage (e.g., 10 A, 250 V AC).
* **Inductive Load Protection**: Fans (motors) are inductive loads. When the relay disconnects the fan, the motor's magnetic field collapses and generates a brief high-voltage spike. A flyback diode across the fan motor terminals (not just the relay coil) provides additional protection.

## Try This! (Challenges)
1. **User-Adjustable Thresholds**: Read a potentiometer on ADC0/GP26 to shift the ON threshold dynamically (e.g., potentiometer maps to 20–35 °C range). Keep the hysteresis band fixed at 2 °C below the new ON threshold for the OFF value.
2. **Speed Control Simulation**: Replace the relay with a PWM signal on the servo pin (GPIO 13) — modulate duty cycle from 0–100% proportionally to the temperature between TEMP_OFF and TEMP_ON + 5 °C, simulating variable-speed fan control via PWM instead of simple on/off switching.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Fan relay never closes | Temperature slider not above 28 °C, or DHT22 read error | Drag slider above 28 °C; check for "DHT22 read error!" in Serial Monitor. |
| Relay chatters rapidly at 27 °C | Hysteresis gap too small or TEMP_ON equals TEMP_OFF | Ensure `TEMP_ON > TEMP_OFF`; the 2 °C gap should prevent chatter. |
| DHT22 read error on every cycle | Missing pull-up resistor or data pin mismatch | Verify `DHTPIN = 12` and that a 10 kΩ pull-up is present on GPIO 12. |
| Green LED does not light | LED_G pin constant not defined or inverted | Confirm `#define LED_G 24` is available or add `#define LED_G 24` at the top of the file. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [163 - Water Quality Station](163-water-quality-station.md)
- [167 - Current Overload Contactor Breaker](167-current-overload-contactor-breaker.md)
- [71 - DHT22 Temp & Humidity Serial Logs](../intermediate/71-dht22-temp-humidity-serial.md)
