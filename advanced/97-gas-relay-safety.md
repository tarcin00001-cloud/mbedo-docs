# 97 - Gas Relay Safety

Build an automatic gas safety shutoff: when an MQ-2 sensor detects dangerous gas levels, a relay cuts the gas solenoid valve, a buzzer sounds, and an LCD displays the alert. Normal operation auto-resumes when levels clear.

## Goal
Learn how to build a hysteresis-controlled safety relay: the relay activates at one threshold and only deactivates when readings fall below a lower threshold, preventing rapid switching (chattering) near the hazard boundary.

## What You Will Build
The system monitors gas concentration continuously:
- **Safe zone (< 400 ADC)**: Relay open (gas supply flowing), Green LED on, LCD shows "GAS: SAFE".
- **Alert zone (400 to 600)**: Yellow LED on, slow warning beep, LCD shows "GAS: WARNING". Relay still open.
- **Danger zone (> 600)**: Relay closes (gas cutoff), Red LED on, rapid siren, LCD shows "GAS: SHUTOFF!". Relay only reopens once reading drops back below 400 (hysteresis gap).

**Why A0, D3, D4, D5, D7, D8?** Pin A0 reads the MQ-2. Pins D3/D4/D5 drive traffic-light LEDs. Pin D7 controls the relay. Pin D8 drives the buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MQ-2 Gas Sensor | `mq2_sensor` | Yes | Yes |
| 5V Relay | `relay` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Yellow) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes (per LED) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| MQ-2 Sensor | VCC | 5V | Power supply |
| MQ-2 Sensor | A0 | A0 | Analog output |
| MQ-2 Sensor | GND | GND | Ground reference |
| Relay | IN | D7 | Control signal |
| Relay | VCC | 5V | Power supply |
| Relay | GND | GND | Ground reference |
| Green LED | Anode (+) | D3 | Safe indicator |
| Green LED | Cathode (-) | GND | Via 220 ohm resistor |
| Yellow LED | Anode (+) | D4 | Warning indicator |
| Yellow LED | Cathode (-) | GND | Via 220 ohm resistor |
| Red LED | Anode (+) | D5 | Danger indicator |
| Red LED | Cathode (-) | GND | Via 220 ohm resistor |
| Buzzer | + | D8 | Signal pin |
| Buzzer | - | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int GAS_PIN    = A0;
const int RELAY_PIN  = 7;
const int GREEN_PIN  = 3;
const int YEL_PIN    = 4;
const int RED_PIN    = 5;
const int BUZ_PIN    = 8;

// Hysteresis thresholds
const int DANGER_ON_THRESHOLD  = 600; // Activate relay (shutoff) above this
const int DANGER_OFF_THRESHOLD = 400; // Deactivate relay (restore) below this
const int WARNING_THRESHOLD    = 400; // Show warning above this

bool relayActive = false; // Track relay state for hysteresis logic

LiquidCrystal_I2C lcd(0x27, 16, 2);

void allLEDsOff() {
  digitalWrite(GREEN_PIN, LOW);
  digitalWrite(YEL_PIN,   LOW);
  digitalWrite(RED_PIN,   LOW);
}

void setup() {
  Serial.begin(9600);
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(YEL_PIN,   OUTPUT);
  pinMode(RED_PIN,   OUTPUT);
  pinMode(BUZ_PIN,   OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW); // Gas valve open by default
  allLEDsOff();
  noTone(BUZ_PIN);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0); lcd.print("Gas Safety Sys  ");
  lcd.setCursor(0, 1); lcd.print("Initializing... ");
  delay(1500);
  lcd.clear();
  
  Serial.println("Gas Relay Safety System Active");
}

void loop() {
  int gasLevel = analogRead(GAS_PIN);
  
  Serial.print("Gas ADC: "); Serial.println(gasLevel);
  
  allLEDsOff();
  
  // Hysteresis relay logic
  if (!relayActive && gasLevel > DANGER_ON_THRESHOLD) {
    relayActive = true;
    digitalWrite(RELAY_PIN, HIGH); // Close relay -> cut gas supply
    Serial.println("!! GAS SHUTOFF ACTIVATED !!");
  } else if (relayActive && gasLevel < DANGER_OFF_THRESHOLD) {
    relayActive = false;
    digitalWrite(RELAY_PIN, LOW); // Open relay -> restore gas supply
    Serial.println("Gas levels normal. Relay released.");
  }
  
  // Drive outputs based on current gas level
  if (relayActive || gasLevel > DANGER_ON_THRESHOLD) {
    // Danger zone
    digitalWrite(RED_PIN, HIGH);
    lcd.setCursor(0, 0); lcd.print("GAS: SHUTOFF!   ");
    lcd.setCursor(0, 1); lcd.print("RELAY CLOSED    ");
    tone(BUZ_PIN, 1000, 300); delay(350);
    tone(BUZ_PIN, 600, 300);  delay(350);
    
  } else if (gasLevel > WARNING_THRESHOLD) {
    // Warning zone
    digitalWrite(YEL_PIN, HIGH);
    noTone(BUZ_PIN);
    lcd.setCursor(0, 0); lcd.print("GAS: WARNING    ");
    lcd.setCursor(0, 1); lcd.print("Level:");
    lcd.print(gasLevel); lcd.print("       ");
    tone(BUZ_PIN, 800, 100);
    delay(2000); // Single slow beep
    
  } else {
    // Safe zone
    digitalWrite(GREEN_PIN, HIGH);
    noTone(BUZ_PIN);
    lcd.setCursor(0, 0); lcd.print("GAS: SAFE       ");
    lcd.setCursor(0, 1); lcd.print("Level:");
    lcd.print(gasLevel); lcd.print("       ");
    delay(500);
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **MQ-2 Sensor**, **Relay**, **three LEDs** (G/Y/R), **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Connect all components per the wiring table.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Slowly drag the MQ-2 slider from 0 up through 400, then above 600. Watch the relay activate. Then drag back below 400 to see the relay release.

## Expected Output

Terminal:
```
Gas Safety System Active
Gas ADC: 150
Gas ADC: 420
Gas ADC: 680
!! GAS SHUTOFF ACTIVATED !!
Gas ADC: 350
Gas levels normal. Relay released.
```

## Expected Canvas Behavior

| MQ-2 Reading | Zone | Relay | Active LED | Buzzer | LCD Line 1 |
| --- | --- | --- | --- | --- | --- |
| 0 to 399 | Safe | Open | Green | Silent | `GAS: SAFE` |
| 400 to 599 | Warning | Open | Yellow | Slow beep | `GAS: WARNING` |
| > 600 | Danger | Closed | Red | Rapid siren | `GAS: SHUTOFF!` |
| < 400 (after shutoff) | Safe | Opens | Green | Silent | `GAS: SAFE` |

## Code Walkthrough

| Expression | What It Does |
| --- | --- |
| `DANGER_ON_THRESHOLD = 600` | Relay activates (gas cuts off) when reading exceeds this value. |
| `DANGER_OFF_THRESHOLD = 400` | Relay only deactivates when reading falls below this lower value. The 200-unit gap is the hysteresis deadband. |
| `relayActive` boolean flag | Tracks relay state independently from the raw sensor value to implement hysteresis. Without it, the relay would chatter at the 600 threshold boundary. |

## Hardware & Safety Concept: Hysteresis in Safety Systems
**Hysteresis** is a deliberate gap between the activation and deactivation threshold of a control system. Without hysteresis:
- At gas level 601, relay closes.
- At gas level 599, relay opens.
- At gas level 601, relay closes again.
- This rapid switching is called **chatter** and destroys relay contacts within hours.

With a 200-unit hysteresis band (close at 600, reopen only below 400), the relay stays closed until levels have clearly and substantially dropped, protecting the relay contacts and preventing false restores.

## Try This! (Challenges)
1. **Manual Override Button**: Wire a button to D2. When pressed in SHUTOFF mode, force the relay open for 60 seconds to allow manual inspection, then re-engage automatic control.
2. **Event Logger**: On each SHUTOFF activation, increment a counter stored in EEPROM. Display the lifetime shutoff count on LCD line 2 during SAFE mode.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay clicks rapidly near threshold | Hysteresis gap too small or missing | Confirm `DANGER_OFF_THRESHOLD` (400) is meaningfully less than `DANGER_ON_THRESHOLD` (600). |
| System starts in SHUTOFF mode | MQ-2 not heated up yet | Physical MQ-2 sensors need 20 to 60 seconds warm-up time. Add a startup delay or ignore readings during the first 60 seconds. |

## Mode Notes
These patterns (analog reads, hysteresis relay logic, multi-LED, and LCD output) are supported by MbedO interpreted mode.

## Related Projects
- [87 - Air Quality Monitor](87-air-quality-monitor.md)
- [88 - Fire Safety System](88-fire-safety-system.md)
- [98 - Proximity Guard](98-proximity-guard.md)
