# 13 - ESP32 5V DC Fan Toggle

Build an automated cooling fan controller that toggles a 5V DC motor/fan via a relay module and updates status messages on an I2C LCD.

## Goal
Learn how to control high-current inductive loads (motors/fans) using relays, configure standard ESP32 I2C pins (GPIO 21/22), and write display status drivers in C++.

## What You Will Build
An automated ventilation system: a 5V DC Fan (motor) is switched ON and OFF by a Relay Module (GPIO 15) every 5 seconds. A 16x2 I2C LCD (GPIO 21, 22) displays the current ventilation state.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| DC Motor (5V Fan) | `dc_motor` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Relay Module | IN (Signal) | GPIO15 | Orange | Relay control input pin |
| Relay Module | VCC / GND | 5V / GND | Red / Black | Relay module power lines |
| DC Motor (Fan) | Terminal 1 | Relay COM | Yellow | Motor power feed line |
| DC Motor (Fan) | Terminal 2 | GND | Black | Motor ground return |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Shared Orange/Blue | Shared I2C Bus |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | Display power lines |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the motor positive terminal to the Relay Common (COM) contact. Connect the Relay Normally Open (NO) contact to the ESP32 5V (VBUS) pin. This routes 5V power to the fan only when the relay is active. The LCD SCL/SDA are connected to GPIO 22 and 21.

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// Hardware pin mappings
const int RELAY_PIN = 15;

// Initialize the 16x2 I2C LCD (address 0x27)
// Default ESP32 I2C pins: SDA = GPIO 21, SCL = GPIO 22
LiquidCrystal_I2C lcd(0x27, 16, 2);

const int TOGGLE_INTERVAL = 5000; // Toggle fan state every 5 seconds

void setup() {
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with fan OFF
  
  Serial.begin(115200);
  
  // Initialize I2C LCD
  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Ventilation Sys");
  lcd.setCursor(0, 1);
  lcd.print("System Ready...");
  delay(1500);
}

void loop() {
  // 1. Turn Fan ON
  digitalWrite(RELAY_PIN, HIGH);
  Serial.println("Fan State: ON");
  
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Fan: RUNNING");
  lcd.setCursor(0, 1);
  lcd.print("State: Cooling  ");
  
  delay(TOGGLE_INTERVAL);
  
  // 2. Turn Fan OFF
  digitalWrite(RELAY_PIN, LOW);
  Serial.println("Fan State: OFF");
  
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Fan: STOPPED");
  lcd.setCursor(0, 1);
  lcd.print("State: Idle     ");
  
  delay(TOGGLE_INTERVAL);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Relay**, **DC Motor**, and **I2C LCD** onto the canvas.
2. Connect Relay **IN** to **GPIO15**. Connect Relay **COM** to DC Motor **Pin 1**.
3. Connect Relay **NO** to ESP32 **5V** (VBUS). Connect DC Motor **Pin 2** to **GND**.
4. Connect LCD SCL to **GPIO22**, SDA to **GPIO21**, VCC to **5V**, and GND to **GND**.
5. Paste code, select interpreted C++ mode, and click **Run**.
6. Watch the motor spin on the canvas and observe the LCD text updates.

## Expected Output
Serial Monitor:
```
Fan State: ON
Fan State: OFF
```

## Expected Canvas Behavior
* Startup: LCD reads `Ventilation Sys` / `System Ready...` for 1.5 seconds.
* Relay turns ON: DC Motor spins, LCD displays `Fan: RUNNING` / `State: Cooling`.
* 5 seconds: Relay turns OFF: DC Motor stops spinning, LCD displays `Fan: STOPPED` / `State: Idle`.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `LiquidCrystal_I2C lcd(0x27, 16, 2);` | Creates the LCD display object at standard hexadecimal address 0x27. |
| `lcd.init();` | Starts the I2C communication and configures the LCD controller register settings. |
| `digitalWrite(RELAY_PIN, HIGH);` | Drives GPIO 15 HIGH, closing the relay contacts to complete the 5V motor circuit. |

## Hardware & Safety Concept: Inductive Flyback Diodes
Motors are **inductive loads**—they store energy in a magnetic field. When the relay contacts open to cut power, this magnetic field collapses rapidly, creating a high-voltage reverse spike (called **flyback voltage**). On real hardware, this spike can arc across the relay contacts, wearing them down, or cause electrical noise that resets the ESP32. Placing a **1N4007 diode** in parallel with the motor terminals (cathode to positive, anode to negative) absorbs this spike safely.

## Try This! (Challenges)
1. **Temperature-Triggered Fan**: Connect a temperature sensor on GPIO 34. Turn the fan ON only if the measured temperature exceeds 28°C.
2. **Speed Scale indicator**: Wire a Green LED on GPIO 23 that flashes rapidly when the fan is running, and stays OFF when the fan is stopped.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD display is blank or shows white boxes | Contrast adjustment or I2C crash | Turn the blue potentiometer on the back of the physical LCD module to adjust contrast. Ensure the SCL/SDA wires are not reversed. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [11 - ESP32 Relay Module Control](11-esp32-relay-module-control.md)
- [12 - ESP32 Relay Safety Signal Warning](12-esp32-relay-safety-signal-warning.md)
