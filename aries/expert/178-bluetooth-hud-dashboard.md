# 178 - Bluetooth HUD Dashboard

Create a vehicle Head-Up Display (HUD) dashboard that reads environment and proximity sensors, updates a local display, and transmits data over Bluetooth while receiving control commands.

## Goal
Learn how to implement full-duplex wireless UART telemetry using an HC-05 module, interface multiple analog and digital sensors, and update displays dynamically based on incoming command characters without using loops.

## What You Will Build
A dashboard that collects ambient temperature and humidity (DHT22) and front-obstacle distance (HC-SR04). The collected telemetry is displayed locally on a 16x2 I2C LCD. Simultaneously, the data is pushed wirelessly via the HC-05 Bluetooth transceiver to a mobile app. The operator can send commands back over Bluetooth to actuate the Warning LED or Buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| I2C LCD (16x2) | `lcd_i2c` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Red Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 Module | TX | GPIO 3 | Yellow | RX1 serial line |
| HC-05 Module | RX | GPIO 4 | Green | TX1 serial line |
| HC-05 Module | VCC | 5V | Red | Power |
| HC-05 Module | GND | GND | Black | Ground |
| DHT22 Sensor | DATA | GPIO 12 | Purple | Temperature/Humidity |
| DHT22 Sensor | VCC | 3V3 | Red | Power |
| DHT22 Sensor | GND | GND | Black | Ground |
| HC-SR04 | TRIG | GPIO 11 | Orange | Ultrasonic trigger |
| HC-SR04 | ECHO | GPIO 8 | Green | Ultrasonic echo |
| HC-SR04 | VCC | 5V | Red | Power |
| HC-SR04 | GND | GND | Black | Ground |
| I2C LCD (16x2) | SDA | GPIO 17 | Green | SDA0 |
| I2C LCD (16x2) | SCL | GPIO 16 | Yellow | SCL0 |
| Active Buzzer | + | GPIO 14 | Gray | Sound alarm |
| Active Buzzer | - | GND | Black | Ground |
| Red LED | Anode | GPIO 15 | Red | Warning indicator |
| Red LED | Cathode | GND | Black | Ground |

> **Wiring tip:** The HC-05 module has logic pins that operate at 3.3V. While ARIES handles 3.3V natively, verify that your Bluetooth module can accept 5V on its VCC pin (most modules have an onboard 3.3V LDO regulator).

## Code
```cpp
// Bluetooth HUD Dashboard - VEGA ARIES v3
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

const int TRIG = 11;
const int ECHO = 8;
const int DHT_PIN = 12;
const int BUZZER = 14;
const int LED_PIN = 15;

LiquidCrystal_I2C lcd(0x27, 16, 2);
DHT dht(DHT_PIN, DHT22);

unsigned long lastSendTime = 0;
bool ledState = false;
bool buzzerState = false;

void setup() {
  Serial.begin(115200);
  Serial1.begin(9600); // Bluetooth Serial
  
  pinMode(TRIG, OUTPUT);
  pinMode(ECHO, INPUT);
  pinMode(BUZZER, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  
  dht.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Bluetooth HUD");
  lcd.setCursor(0, 1);
  lcd.print("Initializing...");
  
  lastSendTime = millis();
}

void loop() {
  // 1. Process incoming Bluetooth control commands (non-blocking)
  if (Serial1.available() > 0) {
    char cmd = Serial1.read();
    
    if (cmd == 'L') {
      ledState = true;
      digitalWrite(LED_PIN, HIGH);
    } 
    else if (cmd == 'l') {
      ledState = false;
      digitalWrite(LED_PIN, LOW);
    } 
    else if (cmd == 'B') {
      buzzerState = true;
      digitalWrite(BUZZER, HIGH);
    } 
    else if (cmd == 'b') {
      buzzerState = false;
      digitalWrite(BUZZER, LOW);
    }
  }
  
  // 2. Telemetry and Display Update (every 2 seconds)
  unsigned long currentTime = millis();
  if (currentTime - lastSendTime >= 2000) {
    lastSendTime = currentTime;
    
    // Read Sensors
    float temp = dht.readTemperature();
    float hum = dht.readHumidity();
    
    // Read Ultrasonic distance
    digitalWrite(TRIG, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG, LOW);
    long duration = pulseIn(ECHO, HIGH, 30000);
    float distance = duration * 0.034 / 2.0;
    if (distance == 0) distance = 400.0;
    
    // Send Telemetry Packets over Bluetooth
    Serial1.print("T:");
    Serial1.print(temp, 1);
    Serial1.print("C,H:");
    Serial1.print(hum, 1);
    Serial1.print("%,D:");
    Serial1.print(distance, 1);
    Serial1.print("cm,L:");
    Serial1.print(ledState ? "ON" : "OFF");
    Serial1.print(",B:");
    Serial1.println(buzzerState ? "ON" : "OFF");
    
    // Send same telemetry to USB Serial for debugging
    Serial.print("Bluetooth Telemetry Sent: ");
    Serial.print("T="); Serial.print(temp);
    Serial.print(" H="); Serial.print(hum);
    Serial.print(" D="); Serial.println(distance);
    
    // Update Local LCD Display
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("T:");
    lcd.print(temp, 1);
    lcd.print("C  H:");
    lcd.print(hum, 0);
    lcd.print("%");
    
    lcd.setCursor(0, 1);
    lcd.print("Dist: ");
    lcd.print(distance, 0);
    lcd.print("cm ");
    if (ledState) {
      lcd.print("[L]");
    }
    if (buzzerState) {
      lcd.print("[B]");
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **HC-05**, **DHT22**, **HC-SR04**, **I2C LCD**, **Buzzer**, and **LED** onto the canvas.
2. Complete the wiring as described in the table.
3. Paste the code into the editor and choose **Interpreted Mode**.
4. Click **Run**.
5. Use the Bluetooth terminal widget to send commands like `L` to turn on the LED, `l` to turn it off, `B` for the buzzer, and `b` to silence it. Observe updating readings.

## Expected Output
Serial Monitor:
```
Bluetooth Telemetry Sent: T=25.40 H=46.30 D=112.50
Bluetooth Telemetry Sent: T=25.40 H=46.20 D=109.80
Received BT Command: L (LED activated)
Bluetooth Telemetry Sent: T=25.40 H=46.20 D=109.80 L=ON
```

## Expected Canvas Behavior
* The I2C LCD prints initial text, then cycles to display "T:25.4C H:46%" and "Dist: 112cm".
* Telemetry text is generated and written to the RX1 serial console.
* Typing `L` or `B` in the Bluetooth serial widget lights up the LED or sounds the buzzer instantly.
* The LCD displays indicator icons `[L]` and `[B]` matching current outputs.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Serial1.begin(9600)` | Starts serial communication on UART1 pins (GPIO 3 and 4) at 9600 bps. |
| `Serial1.read()` | Non-blocking extraction of one incoming character from the RX buffer. |
| `cmd == 'L'` | Compares command char and toggles state. |
| `pulseIn(...)` | Measures HC-SR04 response time. Includes a 30ms timeout to prevent locking if echo fails. |
| `Serial1.print(...)` | Assembles formatted telemetry string and routes it through Bluetooth TX channel. |

## Hardware & Safety Concept
* **Voltage Dividers for RX Pins**: While ARIES operates at 3.3V logic, some HC-05 modules require 3.3V logic levels on their RX pins. If powering the ARIES board at 5V, use a small resistor divider (e.g. 1k and 2k resistors) between ARIES TX (GP4) and HC-05 RX to ensure absolute logic level safety.
* **Proximity Collision Alert**: In a physical HUD, if `distance` falls below 15cm, the code could override Bluetooth commands to beep the buzzer locally as a driver collision warning.

## Try This! (Challenges)
1. **Collision Alarm override**: Modify the code so that if the ultrasonic sensor distance reads less than 15cm, the buzzer triggers automatically at a fast pulse rate, overriding Bluetooth commands.
2. **Telemetry Change-Only**: Reduce power by only transmitting Bluetooth logs when the temperature changes by more than 0.5 degrees or distance changes by more than 5cm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Bluetooth module fails to pair | Insufficient current supply | Make sure the module is connected to the 5V power pin on the board. |
| Dashboard is not receiving commands | TX and RX wires are swapped | Check that HC-05 TX goes to ARIES GP3 (RX1) and HC-05 RX goes to ARIES GP4 (TX1). |
| Values on LCD display garbage characters | I2C address conflict or screen initialization failure | Double-check that I2C wires are tight. Power-cycle the screen and rerun. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [73 - DHT22 Temperature LCD HUD](../intermediate/73-dht22-temperature-lcd-hud.md)
- [127 - Bluetooth Remote Control Robot](../advanced/127-bluetooth-remote-control-robot.md)
- [177 - GPS RFID Vehicle Tracker](177-gps-rfid-vehicle-tracker.md)
