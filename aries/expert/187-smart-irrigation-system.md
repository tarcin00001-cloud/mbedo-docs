# 187 - Smart Irrigation System

Automate garden irrigation by combining soil moisture sensing, rain detection, temperature/humidity monitoring, and relay-controlled pump activation into a fully autonomous watering system.

## Goal

Learn how to integrate multiple analog and digital sensors into a coordinated decision-making system that activates a relay-driven water pump only when soil is dry, no rain is detected, and ambient temperature is suitable — all without loops or arrays.

## What You Will Build

The ARIES v3 board reads soil moisture from ADC3, a rain sensor from ADC2, and temperature/humidity from a DHT22 on GPIO 12. Based on configurable thresholds, it drives a relay on GPIO 15 to switch a 12 V submersible pump. All decisions and timing are managed through state variables updated each `loop()` cycle. Status is reported to the Serial Monitor.

## Parts Needed

| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| Soil Moisture Sensor | `soilmoisture` | Yes | Yes |
| Rain Sensor Module (analog) | `rainsensor` | Yes | Yes |
| 5 V Relay Module | `relay` | Yes | Yes |
| Submersible Pump (12 V) | — | No | Yes |
| 10 kΩ Pull-up Resistor | `resistor` | No | Yes |
| 12 V DC Power Supply | — | No | Yes |
| Jumper Wires | — | No | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | VCC | 3V3 | Red | Sensor power |
| DHT22 Sensor | DATA | GPIO 12 | Yellow | Data with 10 kΩ pull-up to 3V3 |
| DHT22 Sensor | GND | GND | Black | Ground |
| Soil Moisture | VCC | 3V3 | Red | Sensor power |
| Soil Moisture | A0 (Analog) | ADC3 (GP29) | Green | Analog soil reading |
| Soil Moisture | GND | GND | Black | Ground |
| Rain Sensor | VCC | 3V3 | Red | Sensor power |
| Rain Sensor | A0 (Analog) | ADC2 (GP28) | Blue | Analog rain reading |
| Rain Sensor | GND | GND | Black | Ground |
| Relay Module | IN | GPIO 15 | Orange | Relay control signal |
| Relay Module | VCC | 5V | Red | Relay coil power |
| Relay Module | GND | GND | Black | Ground |
| Relay Module | COM | Pump + | — | Connect to pump positive terminal |
| Relay Module | NO | 12V Supply + | — | Normally-open contact |

> **Wiring tip:** Never connect the 12 V pump circuit directly to ARIES GPIO. The relay module provides optical or transistor isolation between the 3.3 V control signal and the high-current pump circuit. Ensure your relay module has an active-LOW or active-HIGH input matching the code's `digitalWrite(PIN_RELAY, HIGH)` for pump ON. Most common 5 V relay modules are active-LOW; adjust `PUMP_ON` / `PUMP_OFF` defines accordingly for your hardware.

## Code

```cpp
// 187 - Smart Irrigation System
// Soil moisture + Rain sensor + DHT22 -> Relay pump automation
#include <DHT.h>

#define DHTPIN        12
#define DHTTYPE       DHT22
#define PIN_RELAY     15
#define PIN_SOIL      29    // ADC3 = GP29
#define PIN_RAIN      28    // ADC2 = GP28

// Relay logic — change to LOW if module is active-LOW
#define PUMP_ON   HIGH
#define PUMP_OFF  LOW

// Thresholds (0-1023 scale for 10-bit ADC)
#define SOIL_DRY_THRESHOLD  600   // above = dry soil
#define RAIN_WET_THRESHOLD  400   // below = rain detected
#define MAX_PUMP_TICKS      150   // max pump run ~150 × 2 s = 5 minutes

DHT dht(DHTPIN, DHTTYPE);

int  pumpRunning    = 0;
int  pumpTicks      = 0;
int  tickCount      = 0;
float lastTemp      = 0.0;
float lastHumidity  = 0.0;
int  lastSoil       = 0;
int  lastRain       = 0;

void setup() {
  Serial.begin(115200);
  pinMode(PIN_RELAY, OUTPUT);
  digitalWrite(PIN_RELAY, PUMP_OFF);
  dht.begin();
  Serial.println("Smart Irrigation System Initialized.");
  Serial.println("Thresholds: Soil>600=dry  Rain<400=raining  Pump max=5min");
}

void loop() {
  tickCount++;

  // Read sensors every 2 seconds (100 ticks × 20 ms = 2 s)
  if (tickCount >= 100) {
    tickCount = 0;

    lastHumidity = dht.readHumidity();
    lastTemp     = dht.readTemperature();
    lastSoil     = analogRead(PIN_SOIL);
    lastRain     = analogRead(PIN_RAIN);

    Serial.print("Soil:");
    Serial.print(lastSoil);
    Serial.print("  Rain:");
    Serial.print(lastRain);

    if (!isnan(lastTemp) && !isnan(lastHumidity)) {
      Serial.print("  Temp:");
      Serial.print(lastTemp, 1);
      Serial.print("C  Hum:");
      Serial.print(lastHumidity, 1);
      Serial.print("%");
    }

    int soilDry    = (lastSoil > SOIL_DRY_THRESHOLD);
    int raining    = (lastRain < RAIN_WET_THRESHOLD);
    int tooHot     = (!isnan(lastTemp) && lastTemp > 40.0);

    // Decision: pump ON only if dry, not raining, and temperature OK
    if (soilDry && !raining && !tooHot && !pumpRunning) {
      pumpRunning = 1;
      pumpTicks   = 0;
      digitalWrite(PIN_RELAY, PUMP_ON);
      Serial.print("  >> PUMP ON");
    }

    // Stop pump if soil is now wet, rain detected, or max time elapsed
    if (pumpRunning) {
      pumpTicks++;
      if (!soilDry || raining || pumpTicks >= MAX_PUMP_TICKS) {
        pumpRunning = 0;
        pumpTicks   = 0;
        digitalWrite(PIN_RELAY, PUMP_OFF);
        Serial.print("  >> PUMP OFF");
      }
    }

    Serial.println();
  }

  delay(20);
}
```

## What to Click in MbedO

1. Drag **VEGA ARIES v3**, **DHT22**, **Soil Moisture**, **Rain Sensor**, and **Relay Module** onto the canvas.
2. Wire DHT22 **DATA → GPIO 12**, **VCC → 3V3**, **GND → GND**.
3. Wire Soil Moisture **A0 → ADC3 (GP29)**, **VCC → 3V3**, **GND → GND**.
4. Wire Rain Sensor **A0 → ADC2 (GP28)**, **VCC → 3V3**, **GND → GND**.
5. Wire Relay **IN → GPIO 15**, **VCC → 5V**, **GND → GND**.
6. Paste the code and select **Interpreted Mode**.
7. Click **Run**.
8. Set the Soil Moisture slider to a high value (>600) to simulate dry soil — pump should activate.
9. Lower the Rain Sensor slider below 400 to simulate rain — pump should stop.
10. Raise DHT22 temperature above 40°C — pump will be suppressed for heat protection.

## Expected Output

Serial Monitor:
```
Smart Irrigation System Initialized.
Thresholds: Soil>600=dry  Rain<400=raining  Pump max=5min
Soil:650  Rain:750  Temp:24.5C  Hum:55.0%  >> PUMP ON
Soil:380  Rain:750  Temp:24.5C  Hum:55.0%  >> PUMP OFF
Soil:200  Rain:300  Temp:24.5C  Hum:80.0%
```

## Expected Canvas Behavior

* With soil slider high and rain slider high, the Relay widget toggles ON and its LED indicator lights.
* Moving soil slider low or rain slider low turns the relay OFF.
* DHT22 temperature slider above 40°C prevents pump activation regardless of soil state.
* All status lines appear in the Serial Monitor every 2 seconds.

## Code Walkthrough

| Line | Check / Action |
| --- | --- |
| `#define SOIL_DRY_THRESHOLD 600` | ADC value above which soil is considered dry (0–1023 range). |
| `#define RAIN_WET_THRESHOLD 400` | ADC value below which rain is considered detected. |
| `tickCount >= 100` | Sensor poll gate: 100 ticks × 20 ms = 2-second read interval. |
| `analogRead(PIN_SOIL)` | Reads GP29 (ADC3) 10-bit value; higher = drier (resistive sensor). |
| `soilDry && !raining && !tooHot` | All three conditions must be true before the pump is allowed to start. |
| `pumpTicks >= MAX_PUMP_TICKS` | Hard maximum run-time safety: 150 ticks × 2 s = 5-minute cutoff. |
| `digitalWrite(PIN_RELAY, PUMP_ON)` | Activates relay coil, closing the NC/NO contact to power the pump. |

## Hardware & Safety Concept

**Multi-Sensor Decision Fusion:** This project demonstrates sensor fusion — combining inputs from three independent sensors to reach a single actuation decision. No single sensor alone is sufficient. Dry soil might still indicate full water tank if a rain event is imminent; high temperature could indicate the pump would overheat. By AND-ing conditions, the system achieves robustness against false positives.

**Relay Safety:** A relay provides galvanic isolation between the low-voltage microcontroller circuit and the mains or 12 V pump circuit. The relay coil draws ~70 mA at 5 V, which exceeds ARIES GPIO current limits (~8 mA max per pin). A relay module includes a transistor driver and flyback diode to safely switch the coil from a GPIO logic signal.

**Timer Safety:** The `MAX_PUMP_TICKS` limit prevents the pump from running indefinitely if a sensor becomes stuck in a "dry" reading due to a fault, protecting both the water supply and the pump motor from dry-running damage.

## Try This! (Challenges)

1. **Moisture-proportional run time**: Instead of a fixed threshold, calculate pump run duration proportional to how dry the soil is — the drier, the longer it runs. Use `pumpTicks` limit scaled from the soil ADC reading.
2. **Scheduled watering window**: Add a simple tick-based clock that only allows the pump to run between simulated "06:00" and "08:00" (represented by a tick counter rollover), preventing midday watering heat stress on plants.

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Pump never turns on | Thresholds not met simultaneously | Lower SOIL_DRY_THRESHOLD or raise RAIN_WET_THRESHOLD in the defines to calibrate to your sensor. |
| Pump turns on briefly then immediately off | pumpTicks increments too fast | The pump tick is updated every 2-second tick; ensure only the inner `if (pumpRunning)` block increments `pumpTicks`. |
| DHT22 reads NaN | Sensor not wired / missing pull-up | Check GPIO 12 connection and add 10 kΩ between DATA and 3V3 on bare sensors. |
| Relay chatters rapidly | Sensor ADC readings fluctuate near threshold | Add ±20 ADC unit hysteresis: only set soilDry when soil > (THRESHOLD+20) and clear when soil < (THRESHOLD-20). |

## Mode Notes

This project runs in MbedO **interpreted C++ mode**.

## Related Projects

- [186 - Voice Command Controller](186-voice-command-controller.md)
- [188 - Earthquake Early Warning](188-earthquake-early-warning.md)
- [194 - Crop Health Analyzer](194-crop-health-analyzer.md)
