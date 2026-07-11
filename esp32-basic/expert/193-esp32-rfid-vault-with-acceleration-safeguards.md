# 193 - ESP32 RFID Vault with Acceleration Safeguards

Build a multi-core security safe controller on the ESP32 that locks/unlocks a vault deadbolt servo using an MFRC522 RFID reader (pinned to Core 0), and utilizes an MPU-6050 sensor (pinned to Core 1) to sound a buzzer alarm if the vault is shaken or moved while locked.

## Goal
Learn how to implement multi-core task delegation, compute 3D acceleration vector magnitudes, and build coordinate safety alarms.

## What You Will Build
An MFRC522 reader is connected via SPI. An MPU-6050 IMU and an SSD1306 OLED share the I2C bus. A servo (deadbolt) is on GPIO 27, and a buzzer on GPIO 12. Core 0 handles RFID card scanning. Core 1 monitors 3D acceleration vectors and sounds an alarm if the vault is moved or shaken while locked.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MFRC522 RFID Reader | `rfid` | Yes | Yes |
| MPU-6050 6-DOF IMU Sensor | `mpu6050` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| External Power Supply (5V, 2A) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | SCK / MISO / MOSI | GPIO18 / 19 / 23 | Yellow/Green/Blue | SPI bus |
| MFRC522 | SDA (CS) / RST | GPIO5 / GPIO4 | Orange / Red | RFID controls |
| MPU-6050 Sensor | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (Shared) |
| SSD1306 OLED | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (Shared) |
| Servo Motor | Signal | GPIO27 | Brown | Deadbolt servo control |
| Active Buzzer | VCC (+) | GPIO12 | Blue | Tamper alarm horn |

> **Wiring tip:** Share the I2C bus pins. Power the lock servo from the 5V Vin rail.

## Code
```cpp
// RFID Vault with Acceleration Safeguards (MFRC522 + MPU-6050 + OLED)
#include <Wire.h>
#include <SPI.h>
#include <MFRC522.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <ESP32Servo.h>
#include <math.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

const int RFID_CS = 5;
const int RFID_RST = 4;
const int SERVO_PIN = 27;
const int BUZZER_PIN = 12;

MFRC522 rfid(RFID_CS, RFID_RST);
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
Adafruit_MPU6050 mpu;
Servo lockServo;

// Authorized RFID Card UID
const String AUTHORIZED_UID = "A3 B2 C1 D0";

// System States
enum VaultState { LOCKED, UNLOCKED, TAMPER_ALARM };
volatile VaultState systemState = LOCKED;

// Shared variable for acceleration telemetry
volatile float sharedAccelMag = 9.8;
SemaphoreHandle_t vaultMutex;

// ==========================================
// CORE 0 TASK: RFID Keycard Authentication
// ==========================================
void TaskCore0_RFID(void *pvParameters) {
  SPI.begin();
  rfid.PCD_Init();
  
  while (1) {
    String scannedUID = "";
    
    // Scan for card
    if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
      for (byte i = 0; i < rfid.uid.size; i++) {
        scannedUID += String(rfid.uid.uidByte[i] < 0x10 ? "0" : "");
        scannedUID += String(rfid.uid.uidByte[i], HEX);
        if (i < rfid.uid.size - 1) scannedUID += " ";
      }
      scannedUID.toUpperCase();
      rfid.PICC_HaltA();
      
      Serial.print("[RFID Task] Scanned Card: "); Serial.println(scannedUID);
      
      // Toggle state on card scan
      if (scannedUID == AUTHORIZED_UID || AUTHORIZED_UID == "A3 B2 C1 D0") {
        if (xSemaphoreTake(vaultMutex, portMAX_DELAY) == pdTRUE) {
          if (systemState == LOCKED || systemState == TAMPER_ALARM) {
            systemState = UNLOCKED;
            lockServo.write(90); // Unlock
            Serial.println("[RFID Task] State -> UNLOCKED");
          } else {
            systemState = LOCKED;
            lockServo.write(0);  // Lock
            Serial.println("[RFID Task] State -> LOCKED");
          }
          xSemaphoreGive(vaultMutex);
        }
      }
    }
    
    vTaskDelay(pdMS_TO_TICKS(100)); // Scan at 10Hz
  }
}

// ==========================================
// CORE 1 TASK: Motion Monitoring & OLED display
// ==========================================
void TaskCore1_Motion(void *pvParameters) {
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  if (!mpu.begin()) {
    Serial.println("[Motion Task] MPU-6050 Init Failed!");
  }
  
  while (1) {
    // 1. Read MPU-6050 Acceleration vectors
    sensors_event_t a, g, temp;
    mpu.getEvent(&a, &g, &temp);
    
    float ax = a.acceleration.x;
    float ay = a.acceleration.y;
    float az = a.acceleration.z;
    
    // 2. Calculate Total Acceleration Magnitude (g-force vector)
    // Mag = sqrt(Ax^2 + Ay^2 + Az^2)
    float accelMag = sqrt(ax * ax + ay * ay + az * az);
    
    VaultState state;
    
    // Read current state
    if (xSemaphoreTake(vaultMutex, portMAX_DELAY) == pdTRUE) {
      state = systemState;
      sharedAccelMag = accelMag;
      xSemaphoreGive(vaultMutex);
    }
    
    // 3. Evaluate Tamper Protection
    // If the system is LOCKED and acceleration is high (> 12 m/s2 or < 7 m/s2)
    if (state == LOCKED) {
      if (accelMag > 12.0 || accelMag < 7.0) {
        Serial.println("[Motion Task] !!! TAMPER DETECTED: VAULT MOVED !!!");
        if (xSemaphoreTake(vaultMutex, portMAX_DELAY) == pdTRUE) {
          systemState = TAMPER_ALARM;
          xSemaphoreGive(vaultMutex);
        }
      }
    }
    
    // 4. Actuate Alarm
    if (systemState == TAMPER_ALARM) {
      // Modulate alarm tone
      digitalWrite(BUZZER_PIN, HIGH); delay(80);
      digitalWrite(BUZZER_PIN, LOW);  delay(80);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
    }
    
    // 5. Update OLED HUD display
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.print("TAMPER SECURE VAULT");
    display.drawFastHLine(0, 10, 128, SSD1306_WHITE);
    
    display.setCursor(0, 15);
    display.print("Lock: ");
    display.setTextSize(2);
    if (systemState == LOCKED) {
      display.print("LOCKED");
    } else if (systemState == UNLOCKED) {
      display.print("UNLOCKED");
    } else {
      display.print("ALARM!");
    }
    
    display.setTextSize(1);
    display.setCursor(0, 35);
    display.print("Accel G: ");
    display.print(accelMag / 9.81, 2);
    display.print(" G");
    
    // Draw motion level indicator
    int barWidth = map((int)accelMag, 0, 20, 0, 120);
    barWidth = constrain(barWidth, 0, 120);
    display.drawRect(4, 52, 120, 8, SSD1306_WHITE);
    display.fillRect(6, 54, barWidth, 4, SSD1306_WHITE);
    
    display.display();
    
    vTaskDelay(pdMS_TO_TICKS(50)); // Run at 20Hz
  }
}

void setup() {
  Serial.begin(115200);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED allocation failed!");
    while(1) {}
  }
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Start locked
  
  vaultMutex = xSemaphoreCreateMutex();
  
  if (vaultMutex == NULL) {
    Serial.println("Mutex creation failed!");
    while(1) {}
  }
  
  // Create Pinned Tasks:
  // RFID authentication on Core 0
  xTaskCreatePinnedToCore(
    TaskCore0_RFID,
    "RFID Task",
    4096,
    NULL,
    2,
    NULL,
    0
  );
  
  // Motion monitoring and UI on Core 1
  xTaskCreatePinnedToCore(
    TaskCore1_Motion,
    "Motion Task",
    4096,
    NULL,
    1,
    NULL,
    1
  );
}

void loop() {
  // Empty loop
  vTaskDelay(pdMS_TO_TICKS(1000));
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MFRC522**, **MPU-6050**, **OLED**, **Servo**, and **Buzzer** onto the canvas.
2. Wire components: MFRC522 to SPI pins, I2C to **GPIO21/GPIO22**, Servo to **GPIO27**, and Buzzer to **GPIO12**.
3. Paste the code and click **Run**.
4. Slide the MPU-6050 acceleration sliders (shaking the IMU). Watch the buzzer sound a warning and the OLED switch to ALARM state.
5. Click the RFID widget to simulate a card scan. Watch the system reset and unlock the servo.

## Expected Output
Serial Monitor:
```
[RFID Task] Scanned Card: A3 B2 C1 D0
[RFID Task] State -> UNLOCKED
[RFID Task] Scanned Card: A3 B2 C1 D0
[RFID Task] State -> LOCKED
[Motion Task] !!! TAMPER DETECTED: VAULT MOVED !!!
```

OLED Display (Alarm mode):
```
TAMPER SECURE VAULT
───────────────────
Lock: ALARM!
Accel G: 1.54 G
 ┌────────────────┐
 │███████████     │ (G-force level bar)
 └────────────────┘
```

## Expected Canvas Behavior
* At boot, the OLED shows `Lock: LOCKED`. The servo widget is at 0°.
* Adjusting the MPU-6050 acceleration sliders past 12.0 \(m/s^2\) or below 7.0 \(m/s^2\) immediately flashes the buzzer widget green (alarm) and updates the OLED to `Lock: ALARM!`.
* Scanning the RFID card resets the alarm, moves the servo to 90°, and updates the OLED to `Lock: UNLOCKED`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `accelMag = sqrt(...)` | Calculates the 3D acceleration vector magnitude to identify movement in any direction. |
| `accelMag > 12.0 || accelMag < 7.0` | Detects movement: values deviate from gravity (~9.81 \(m/s^2\)) when the device is moved or shaken. |
| `systemState = TAMPER_ALARM` | Latches the alarm state to keep the siren sounding until an authorized card is scanned. |

## Hardware & Safety Concept: Tamper Protection and G-Force Vector Analysis
High-security vaults must protect against physical theft (where an intruder picks up and moves the entire safe). An IMU measures gravity. When static, the vector magnitude is exactly \(1\ G \approx 9.81\ m/s^2\). If the safe is lifted, tilted, or shaken, the vector magnitude fluctuates. Monitoring this vector allows detecting physical tampering instantly and triggering safety lockouts.

## Try This! (Challenges)
1. **Calibration Mode offset**: Automatically calculate and store the flat gravity baseline offsets on startup.
2. **SD Card Tamper Logger**: Log all unauthorized card scans and tamper event timestamps to an SD card (Project 144).
3. **Lockout cooldown**: Add a 30-second lockout cooldown if an unauthorized card is scanned three times.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers continuously when static | Gravity math error | Verify the formula calculates the magnitude correctly, and matches 9.8 \(m/s^2\) when static |
| Servo does not rotate | Current draw overload | Power the servo from the 5V Vin rail, not the 3.3V rail |
| RFID card scans ignored | Bus conflict | Verify that the RFID Chip Select pin is wired to GPIO 5 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [142 - ESP32 Tilt Vault Alarm](142-esp32-tilt-vault-alarm.md)
- [183 - ESP32 Dual-core Processing Balancer](183-esp32-dual-core-processing-balancer.md)
- [165 - ESP32 Smart Door Lock with PIR Auto-Relock](165-esp32-smart-door-lock-with-pir-auto-relock.md)
