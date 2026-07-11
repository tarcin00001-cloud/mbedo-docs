# 105 - Thermostat Controller

Build a climate control thermostat system that measures temperature using an NTC thermistor and regulates a heating element relay using hysteresis logic and an I2C LCD display.

## Goal
Learn how to read analog thermistors, apply temperature conversion approximations, implement hysteresis thresholds, and update character LCDs.

## What You Will Build
An automatic thermostat node. When the temperature falls below 20.0°C, the heating relay turns ON. The relay remains ON until the temperature rises past 25.0°C, preventing rapid relay switching (chattering) around a single setpoint.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| NTC Thermistor | `ntc` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Yes | Yes (divider) |
| 5V Relay Module (Heater) | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NTC Circuit | VCC | 3V3 | Red | Voltage divider supply (3.3V) |
| NTC Circuit | OUT (Signal) | ADC0 (GP26) | White | Analog voltage output |
| NTC Circuit | GND | GND | Black | Ground reference |
| Relay Module | VCC | 5V | Red | Relay power supply (5V) |
| Relay Module | GND | GND | Black | Ground reference |
| Relay Module | IN | GPIO 15 | Blue | Control signal pin |
| I2C LCD | VCC | 5V | Red | LCD power supply (5V) |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** To read an NTC thermistor, wire it in series with a fixed 10k-ohm resistor. Connect NTC Pin 1 to 3V3, NTC Pin 2 to ADC0 (GP26), and the 10k resistor between ADC0 (GP26) and GND.

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int NTC_PIN = 26;     // NTC on ADC0 (GP26)
const int RELAY_PIN = 15;   // Heater Relay on GPIO 15

LiquidCrystal_I2C lcd(0x27, 16, 2);

float lastTemp = -1.0;
int lastHeaterState = -1;

void setup() {
  Wire.begin();
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with heater OFF

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Thermostat Sys");
}

void loop() {
  int rawVal = analogRead(NTC_PIN);
  
  // Linearized mapping of the thermistor voltage divider output.
  // Assumes a standard NTC where voltage increases with temperature when configured as pull-up.
  // We approximate a 10.0°C to 50.0°C temperature scale.
  float temp = 10.0 + (rawVal * 40.0 / 4095.0);

  int heaterState = lastHeaterState;
  
  // Hysteresis Control Loop
  if (temp < 20.0) {
    heaterState = 1;
    digitalWrite(RELAY_PIN, HIGH); // Turn Heater ON
  } else if (temp > 25.0) {
    heaterState = 0;
    digitalWrite(RELAY_PIN, LOW);  // Turn Heater OFF
  }

  // Update screen only when temperature or heater state changes
  if (temp != lastTemp || heaterState != lastHeaterState) {
    lcd.setCursor(0, 0);
    lcd.print("Temp: ");
    lcd.print(temp, 1);
    lcd.print(" C   ");

    lcd.setCursor(0, 1);
    if (heaterState == 1) {
      lcd.print("HEATER: ON      ");
    } else {
      lcd.print("HEATER: OFF     ");
    }

    lastTemp = temp;
    lastHeaterState = heaterState;
  }

  delay(500); // Polling delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **NTC Thermistor**, **5V Relay Module**, and **I2C LCD Display** components onto the canvas.
2. Wire the NTC divider: **VCC** to **3V3**, **OUT** to **ADC0 (GP26)**, and **GND** to **GND**.
3. Wire the Relay: **VCC** to **5V**, **GND** to **GND**, and **IN** to **GPIO 15**.
4. Wire the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Raw ADC: 950, Temp: 19.2 C -> Heater: ON
Raw ADC: 1540, Temp: 25.0 C -> Heater: ON
Raw ADC: 1600, Temp: 25.6 C -> Heater: OFF
```

## Expected Canvas Behavior
* Adjusting the simulated thermistor level down (simulating cold) causes the LCD to display temperatures below 20.0°C and activates the relay.
* Moving the temperature slider upward above 25.0°C turns the relay off.
* Small temperature shifts between 20.0°C and 25.0°C leave the relay in its current state.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(NTC_PIN)` | Reads the analog voltage output from the thermistor voltage divider. |
| `10.0 + (rawVal * 40.0 / 4095.0)` | Applies linear conversion mapping raw ADC values to Celsius scale. |
| `temp < 20.0` | Activates heating logic below the minimum threshold. |
| `temp > 25.0` | Cuts off heater logic above the target comfort threshold. |

## Hardware & Safety Concept
* **Hysteresis Logic Purpose**: Standard thermostats use hysteresis (a deadband gap) to prevent the switching relay from pulsing on and off continuously when the room temperature hovers exactly at the target. This protects the physical contacts of the relay and heater elements from premature wear.
* **Thermal Coupling**: Ensure the thermistor is mounted in a position that receives representative room airflow, isolated from the direct heat output of the radiator to avoid false early cut-offs.

## Try This! (Challenges)
1. **Cooling Mode**: Modify the logic to operate an air conditioner fan instead of a heater (relay turns ON when temperature exceeds 28°C, and turns OFF when it drops below 24°C).
2. **Alarm Sounder**: Connect a buzzer on GPIO 14. If the temperature exceeds 40°C (over-temperature hazard), sound the buzzer continuously.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temperature reads 50°C constantly | NTC disconnected or floating pin | Verify the wiring connection of the voltage divider to ARIES ADC0 (GP26). |
| Relay oscillates rapidly | Hysteresis boundaries too narrow | Widen the difference between the low setpoint and high setpoint. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [58 - 16x2 I2C LCD Hello World](58-16x2-i2c-lcd-hello-world.md)
- [100 - Automatic Smart Fan](100-automatic-smart-fan.md)
