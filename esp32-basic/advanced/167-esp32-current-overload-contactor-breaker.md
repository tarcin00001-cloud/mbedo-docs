# 167 - ESP32 Current Overload Contactor Breaker

Build an industrial-grade electronic circuit breaker (contactor protector) that measures load current using an ACS712 sensor, trips a safety relay (opening the contactor) if the current exceeds 1.5A, sounds a buzzer alarm, and requires a manual button reset to restore power.

## Goal
Learn how to measure AC/DC currents using Hall effect sensors, implement fast-trip overcurrent protection logic with safety latch states, and build a contactor reset loop.

## What You Will Build
An ACS712 current sensor is connected to GPIO 34. A relay (contactor) is on GPIO 13, a buzzer on GPIO 4, a push button on GPIO 12, and a 16x2 LCD on I2C. The relay is closed by default. If the load current exceeds 1500 mA, the relay opens instantly, the buzzer sounds, and the LCD displays "OVERLOAD TRIP". The user must press the button to reset the breaker.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| ACS712 Current Sensor (5A) | `acs712` | Yes | Yes |
| Relay Module (Contactor) | `relay` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| 10 kΩ Resistor (for button) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ACS712 Sensor | OUT (Analog) | GPIO34 | Yellow | Current signal input |
| Relay Module | IN (Signal) | GPIO13 | Orange | Load contactor control |
| Push Button | Pin 1 / Pin 2 | GPIO12 / 3V3 | Green / Red | Reset button (active-HIGH) |
| Push Button | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Active Buzzer | VCC (+) | GPIO4 | Blue | Alarm siren |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the push button with an external 10 kΩ pull-down resistor to GPIO 12 to guarantee a stable LOW state when open.

## Code
```cpp
// Current Overload Contactor Breaker (ACS712 + Relay + Buzzer + Button)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int CURRENT_PIN = 34;
const int RELAY_PIN = 13;
const int BUTTON_PIN = 12;
const int BUZZER_PIN = 4;

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Breaker Settings
const float CURRENT_LIMIT_MA = 1500.0; // Overcurrent trip threshold (1.5 A)
const float ACS_OFFSET = 1.65;         // Zero-current voltage offset (3.3V / 2)
const float ACS_SENSITIVITY = 0.185;   // 185 mV/A for 5A ACS712 sensor

// System States
enum BreakerState { ACTIVE, TRIPPED };
BreakerState systemState = ACTIVE;

unsigned long lastSampleTime = 0;
const unsigned long SAMPLE_INTERVAL_MS = 200; // Sample current every 200 ms

float readCurrentmA() {
  long sum = 0;
  const int NUM_SAMPLES = 50;
  
  // Average multiple samples to filter out electrical noise and spikes
  for (int i = 0; i < NUM_SAMPLES; i++) {
    sum += analogRead(CURRENT_PIN);
    delayMicroseconds(100);
  }
  float avgRaw = (float)sum / NUM_SAMPLES;
  float voltage = avgRaw * (3.3 / 4095.0);
  
  // Convert voltage difference to current in Amps
  float currentA = (voltage - ACS_OFFSET) / ACS_SENSITIVITY;
  
  // Ignore minor noise offsets around 0
  if (abs(currentA) < 0.04) currentA = 0.0;
  
  return currentA * 1000.0; // Return current in milliamperes
}

void setup() {
  Serial.begin(115200);
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT); // External pull-down wired
  pinMode(BUZZER_PIN, OUTPUT);
  
  // Initialize relay closed (ON - supplying load power)
  digitalWrite(RELAY_PIN, HIGH);
  digitalWrite(BUZZER_PIN, LOW);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Breaker");
  lcd.setCursor(0, 1);
  lcd.print("System Active");
  
  delay(1500);
  lcd.clear();
}

void loop() {
  float currentmA = readCurrentmA();
  bool resetButton = (digitalRead(BUTTON_PIN) == HIGH);
  unsigned long now = millis();
  
  // Breaker Logic
  switch (systemState) {
    case ACTIVE:
      // Turn relay closed (ON)
      digitalWrite(RELAY_PIN, HIGH);
      digitalWrite(BUZZER_PIN, LOW);
      
      // Update telemetry display periodically
      if (now - lastSampleTime >= SAMPLE_INTERVAL_MS) {
        lcd.setCursor(0, 0);
        lcd.print("Load: ACTIVE    ");
        lcd.setCursor(0, 1);
        lcd.print("Current: ");
        lcd.print(abs(currentmA), 0);
        lcd.print(" mA   ");
        
        Serial.print("Current: "); Serial.print(currentmA, 1); Serial.println(" mA (SAFE)");
        lastSampleTime = now;
      }
      
      // Overcurrent evaluation
      if (abs(currentmA) >= CURRENT_LIMIT_MA) {
        Serial.println("!!! OVERCURRENT DETECTED: TRIPPING BREAKER !!!");
        // Open contacts instantly (OFF)
        digitalWrite(RELAY_PIN, LOW); 
        digitalWrite(BUZZER_PIN, HIGH); // Alarm ON
        
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("BREAKER TRIPPED ");
        lcd.setCursor(0, 1);
        lcd.print("OVERLOAD TRIP!  ");
        
        systemState = TRIPPED;
      }
      break;
      
    case TRIPPED:
      // Ensure contacts remain open and buzzer sounds
      digitalWrite(RELAY_PIN, LOW);
      
      // Flash display and modulate alarm pitch
      digitalWrite(BUZZER_PIN, HIGH); delay(100);
      digitalWrite(BUZZER_PIN, LOW);  delay(100);
      
      lcd.setCursor(0, 0);
      lcd.print("OVERLOAD: ");
      lcd.print(abs(currentmA), 0);
      lcd.print(" mA   ");
      lcd.setCursor(0, 1);
      lcd.print("PRESS BUTTON RST");
      
      // Check reset switch
      if (resetButton) {
        Serial.println("Reset command received. Cooling down...");
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Resetting...");
        lcd.setCursor(0, 1);
        lcd.print("Cooling contact ");
        
        digitalWrite(BUZZER_PIN, LOW);
        delay(2000); // 2-second safety delay to allow load to cool down
        
        // Reset state
        systemState = ACTIVE;
        lcd.clear();
      }
      break;
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **ACS712**, **Relay**, **Button**, **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Wire components: ACS712 to **GPIO34**, Relay to **GPIO13**, Button to **GPIO12**, Buzzer to **GPIO4**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the current slider on the ACS712 widget past 1.5A. Watch the relay turn OFF and the buzzer sound an alarm.
5. Reduce the current slider below 1.5A, then click the button widget. Watch the system reset and restore power.

## Expected Output
Serial Monitor:
```
Current: 450.0 mA (SAFE)
Current: 1620.0 mA (SAFE)
!!! OVERCURRENT DETECTED: TRIPPING BREAKER !!!
Reset command received. Cooling down...
```

LCD Display (tripped):
```
OVERLOAD: 1620 mA
PRESS BUTTON RST
```

## Expected Canvas Behavior
* At startup, the relay widget turns ON (green).
* Sliding the ACS712 current widget above 1500 mA turns the relay widget OFF (grey) and flashes the buzzer.
* Lowering the current and pressing the button widget turns the relay back ON (green) after 2 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `readCurrentmA()` | Takes an average of 50 samples to filter out transients (like motor startup currents). |
| `abs(currentmA) >= CURRENT_LIMIT_MA` | Evaluates if the current exceeds safe thresholds to trigger a trip. |
| `digitalWrite(RELAY_PIN, LOW)` | Instantly opens the relay contactor to cut off load power. |

## Hardware & Safety Concept: Inrush Currents and Breaker Trip Curves
When an inductive load (like an electric motor) starts up, it draws a temporary surge of current (the **inrush current**) that can be 5 to 10 times higher than its normal running current. If an electronic breaker trips instantly on any spike, it will trip every time the motor starts. To prevent this, the code takes a running average of 50 samples. Real circuit breakers follow a **trip curve** (thermal-magnetic behavior) that allows short-term surges but trips instantly on massive overloads.

## Try This! (Challenges)
1. **Interactive Limit Adjuster**: Add a potentiometer on GPIO 35 to dynamically adjust the trip current threshold from 500 mA to 3000 mA.
2. **SD Card Trip Logger**: Add an SD card module (Project 144) to record every overload trip timestamp to a log file.
3. **Short-Circuit Instant Trip**: Skip the averaging loop if the measured current exceeds 3.0A, triggering an immediate shutdown to protect against short circuits.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Breaker trips immediately at boot | ACS712 offset calibration incorrect | Verify that `ACS_OFFSET` matches half of the supply voltage |
| Button reset does not work | Button floating | Ensure the 10 kΩ pull-down resistor is wired correctly to GPIO 12 |
| Current readings jump around | High electrical noise | Add a capacitor (e.g. 0.1 µF) across the sensor's analog output and ground |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [138 - ESP32 SD Card Current/Energy Logger](138-esp32-sd-card-current-energy-logger.md)
- [153 - ESP32 Temperature PID Heater Control](153-esp32-temperature-pid-heater-control.md)
- [56 - ESP32 Button Controlled Relay Switch](../intermediate/56-esp32-button-controlled-relay-switch.md)
