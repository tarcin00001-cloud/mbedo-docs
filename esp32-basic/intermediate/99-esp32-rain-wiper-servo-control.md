# 99 - ESP32 Rain Wiper Servo Control

Build an automatic windshield wiper system that sweeps a servo motor when rain is detected, adjusting the sweep speed based on rain intensity.

## Goal
Learn how to read analog weather sensors, map rain intensity thresholds to motor sweep speeds, and control a servo's dynamic delay loop.

## What You Will Build
A rain sensor's analog output is connected to GPIO 34. A servo motor on GPIO 13 represents the wiper arm. If dry (ADC high), the servo remains parked at 0°. When water is detected on the sensor pad, the servo begins sweeping back and forth; as the rain gets heavier, the sweep speed increases.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Rain Sensor Module | `rain_sensor` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Rain Sensor | AO | GPIO34 | Yellow | Analog wetness input |
| Rain Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| Servo Motor | Signal | GPIO13 | Orange | Wiper PWM control |
| Servo Motor | VCC | 5V (Vin) | Red | Motor power |
| Servo Motor | GND | GND | Black | Ground reference |

> **Wiring tip:** Connect the rain sensor to **GPIO 34** and the servo control line to **GPIO 13**. Power the servo from 5V (Vin) to supply enough current for the motor sweeps.

## Code
```cpp
// Rain Wiper Servo Control
#include <ESP32Servo.h>

const int RAIN_PIN = 34;
const int SERVO_PIN = 13;

Servo wiperServo;

// Sensor calibration bounds
const int DRY_VAL = 3500; // ADC reading when dry
const int WET_VAL = 1000; // ADC reading when fully wet (heavy rain)

void setup() {
  Serial.begin(115200);
  
  wiperServo.attach(SERVO_PIN);
  wiperServo.write(0); // Park wiper at 0 degrees
  
  Serial.println("Automatic Wiper System ready.");
}

void loop() {
  int rainRaw = analogRead(RAIN_PIN);
  
  // Map raw values to rain percentage (0% to 100%)
  // Invert because lower ADC voltage = wetter sensor
  int rainPct = map(rainRaw, DRY_VAL, WET_VAL, 0, 100);
  rainPct = constrain(rainPct, 0, 100);
  
  Serial.print("Rain raw: "); Serial.print(rainRaw);
  Serial.print(" | Wetness: "); Serial.print(rainPct); Serial.println("%");
  
  // Trigger wiper if wetness is above 15%
  if (rainPct > 15) {
    // Determine sweep speed (delay between steps)
    // Heavy rain (100%) = fast sweep (3ms delay per degree)
    // Light rain (15%) = slow sweep (15ms delay per degree)
    int stepDelay = map(rainPct, 15, 100, 15, 3);
    stepDelay = constrain(stepDelay, 3, 15);
    
    Serial.print("Sweeping with speed step delay: "); Serial.println(stepDelay);
    
    // Sweep Forward 0 to 180
    for (int angle = 0; angle <= 180; angle += 2) {
      wiperServo.write(angle);
      delay(stepDelay);
    }
    
    // Sweep Reverse 180 to 0
    for (int angle = 180; angle >= 0; angle -= 2) {
      wiperServo.write(angle);
      delay(stepDelay);
    }
  } else {
    // Park wiper
    wiperServo.write(0);
    delay(500); // Check once every half second when dry
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Rain Sensor**, and **Servo** onto the canvas.
2. Wire Rain Sensor AO to **GPIO34** and Servo signal to **GPIO13**.
3. Paste the code and click **Run**.
4. Slide the rain sensor widget to simulate water. Watch the servo sweep. The wetter the sensor, the faster the sweeps.

## Expected Output
Serial Monitor:
```
Automatic Wiper System ready.
Rain raw: 3500 | Wetness: 0%
Rain raw: 2200 | Wetness: 52%
Sweeping with speed step delay: 9
Rain raw: 1100 | Wetness: 96%
Sweeping with speed step delay: 3
```

## Expected Canvas Behavior
* While the rain sensor widget is dry, the servo arm stays parked at 0°.
* Wetting the rain sensor widget triggers the servo to sweep back and forth continuously.
* Increasing the wetness makes the servo sweep speed visibly faster on the canvas.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `map(rainRaw, DRY_VAL, WET_VAL, 0, 100)` | Inverted mapping: lower voltage = higher rain intensity %. |
| `map(rainPct, 15, 100, 15, 3)` | Maps wetness percentage to sweep step delay (lower delay = faster rotation). |
| `wiperServo.write(angle)` | Outputs the PWM pulses to sweep the servo. |

## Hardware & Safety Concept: Sensor Lifespans and Electrolysis
Continuous DC voltage applied to wet sensor traces causes rapid galvanic corrosion (electrolysis), eating away the copper plating and destroying the sensor within days. In real projects, only power the rain sensor using a digital GPIO output pin immediately before reading the ADC value, then turn it off between samples.

## Try This! (Challenges)
1. **Appliance Relay Toggle**: Add a relay that turns on a water pump nozzle if a button is pressed (windshield washer fluid).
2. **Rain Alarm Buzzer**: Sound a warning beep on GPIO 15 if rain intensity suddenly shifts past 80% (heavy downpour alert).
3. **Manual Override Switch**: Add a toggle switch that allows the user to turn on the wipers at a fixed speed, ignoring the sensor.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo sweeps once and stops | Reading dropping below threshold during sweep | Ensure the water slider is held active during testing |
| Wipers do not speed up | Mapping limits incorrect | Verify actual `DRY_VAL` and `WET_VAL` using serial logs |
| Motor moves in small steps or halts | Insufficient current | Verify power connects to the ESP32 5V pin |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [39 - ESP32 Rain Sensor Digital Alert](../beginner/39-esp32-rain-sensor-digital-alert.md)
- [51 - ESP32 Servo Motor 180° Sweep](51-esp32-servo-motor-180-sweep.md)
- [41 - ESP32 Water Level Analog Level Print](../beginner/41-esp32-water-level-analog-level-print.md)
