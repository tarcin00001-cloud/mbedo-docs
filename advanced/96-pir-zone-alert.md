# 96 - PIR Zone Alert

Use multiple PIR motion sensors to cover distinct physical zones, identify which zone triggered, and display the zone name on an LCD while sounding a zone-specific alert pattern.

## Goal
Learn how to manage multiple digital inputs representing independent sensor zones, determine which zone fired, and deliver differentiated output responses based on zone identity.

## What You Will Build
Three PIR sensors cover three zones (Front Door, Back Door, Side Window). When motion is detected in any zone:
- The LCD shows which zone was triggered.
- The buzzer plays a zone-specific tone pattern (different tones for each zone).
- The Terminal logs a timestamped zone event.

**Why D2, D3, D4, D8?** Pins D2/D3/D4 read three PIR sensors. Pin D8 drives the buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| PIR Motion Sensor (x3) | `pir_sensor` | Yes (x3) | Yes (x3) |
| Passive Buzzer | `buzzer` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| PIR Sensor 1 (Front) | OUT | D2 | Zone 1 signal |
| PIR Sensor 1 | VCC | 5V | Power supply |
| PIR Sensor 1 | GND | GND | Ground reference |
| PIR Sensor 2 (Back) | OUT | D3 | Zone 2 signal |
| PIR Sensor 2 | VCC | 5V | Power supply |
| PIR Sensor 2 | GND | GND | Ground reference |
| PIR Sensor 3 (Side) | OUT | D4 | Zone 3 signal |
| PIR Sensor 3 | VCC | 5V | Power supply |
| PIR Sensor 3 | GND | GND | Ground reference |
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

// Zone configuration
const int ZONE_COUNT = 3;
const int PIR_PINS[ZONE_COUNT]   = {2, 3, 4};
const int ZONE_TONES[ZONE_COUNT] = {900, 1100, 700}; // Hz per zone
const char* ZONE_NAMES[ZONE_COUNT] = {
  "Front Door ",
  "Back Door  ",
  "Side Window"
};

const int BUZ_PIN = 8;

LiquidCrystal_I2C lcd(0x27, 16, 2);

unsigned long lastTriggerTime = 0;
const unsigned long COOLDOWN_MS = 3000; // 3 second cooldown per zone

void setup() {
  Serial.begin(9600);
  
  for (int i = 0; i < ZONE_COUNT; i++) {
    pinMode(PIR_PINS[i], INPUT);
  }
  pinMode(BUZ_PIN, OUTPUT);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0); lcd.print("PIR Zone Monitor");
  lcd.setCursor(0, 1); lcd.print("All Zones Clear ");
  
  Serial.println("PIR Zone Alert System Active");
  Serial.println("Zones: Front Door | Back Door | Side Window");
}

void loop() {
  unsigned long now = millis();
  
  // Check all zones
  for (int i = 0; i < ZONE_COUNT; i++) {
    if (digitalRead(PIR_PINS[i]) == HIGH) {
      // Cooldown guard: skip if triggered recently
      if (now - lastTriggerTime < COOLDOWN_MS) continue;
      
      lastTriggerTime = now;
      
      Serial.print("[t=");
      Serial.print(now / 1000);
      Serial.print("s] Motion in Zone ");
      Serial.print(i + 1);
      Serial.print(": ");
      Serial.println(ZONE_NAMES[i]);
      
      // Update LCD with zone info
      lcd.setCursor(0, 0);
      lcd.print("Motion Detected!");
      lcd.setCursor(0, 1);
      lcd.print(ZONE_NAMES[i]);
      lcd.print("     ");
      
      // Zone-specific buzzer alert
      tone(BUZ_PIN, ZONE_TONES[i], 400);
      delay(500);
      tone(BUZ_PIN, ZONE_TONES[i] + 200, 200);
      delay(300);
      noTone(BUZ_PIN);
      
      // Return LCD to standby after display
      delay(2000);
      lcd.setCursor(0, 0); lcd.print("PIR Zone Monitor");
      lcd.setCursor(0, 1); lcd.print("All Zones Clear ");
      
      break; // Handle one zone trigger per loop cycle
    }
  }
  
  delay(100); // Polling rate
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **three PIR Sensors**, **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Connect PIR sensors to **D2**, **D3**, **D4**.
3. Connect Buzzer: **+** to **D8**, **-** to **GND**.
4. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.
8. Click each PIR sensor individually and observe the different zone names and tone patterns.

## Expected Output

Terminal:
```
PIR Zone Alert System Active
Zones: Front Door | Back Door | Side Window
[t=5s] Motion in Zone 1: Front Door
[t=12s] Motion in Zone 3: Side Window
```

## Expected Canvas Behavior

| PIR Triggered | Zone Name | Buzzer Tone | LCD Line 2 |
| --- | --- | --- | --- |
| PIR on D2 | Front Door | 900 / 1100 Hz | `Front Door` |
| PIR on D3 | Back Door | 1100 / 1300 Hz | `Back Door` |
| PIR on D4 | Side Window | 700 / 900 Hz | `Side Window` |

Each event shows for 2 seconds, then LCD returns to standby.

## Code Walkthrough

| Expression | What It Does |
| --- | --- |
| `PIR_PINS[]`, `ZONE_TONES[]`, `ZONE_NAMES[]` parallel arrays | Stores per-zone configuration in matching index arrays, so zone `i` always uses `PIR_PINS[i]`, `ZONE_TONES[i]`, and `ZONE_NAMES[i]`. Adding a fourth zone only requires adding one entry to each array. |
| `now - lastTriggerTime < COOLDOWN_MS` | Rate-limits how often the zone alert fires, preventing rapid repeated triggering from a single sustained motion event. |

## Hardware & Safety Concept: Zone-Based Coverage
Commercial alarm systems divide a building into zones for two practical reasons:
1. **Identification**: Knowing which zone triggered helps the operator or monitoring service locate the intrusion quickly.
2. **Independent isolation**: A faulty sensor in one zone can be disabled without disabling the whole system.
Professional panels (like DSC and Honeywell) support 8 to 128+ zones, each with its own wiring run back to a central controller.

## Try This! (Challenges)
1. **Zone Enable/Disable**: Add a 3-position DIP switch on D5, D6, D7. When a switch is OFF, that zone's PIR is bypassed and never triggers an alert.
2. **Event Counter Display**: Track how many times each zone has triggered since startup. Display `Z1:3 Z2:1 Z3:7` on LCD line 2 during standby.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Wrong zone shown on LCD | PIR pin to array index mismatch | Verify that `PIR_PINS[0]` is D2 (Front), `PIR_PINS[1]` is D3 (Back), `PIR_PINS[2]` is D4 (Side). |
| Zone triggers immediately on startup | PIR warm-up period | Physical PIR sensors take 30 to 60 seconds to stabilize after power-on. Add a `delay(30000)` at the end of `setup()` for physical hardware. |

## Mode Notes
These patterns (multi-pin digital reads, array iteration, and LCD output) are supported by MbedO interpreted mode.

## Related Projects
- [95 - BT Alarm ArmDisarm](95-bt-alarm-armdisarm.md)
- [97 - Gas Relay Safety](97-gas-relay-safety.md)
- [88 - Fire Safety System](88-fire-safety-system.md)
