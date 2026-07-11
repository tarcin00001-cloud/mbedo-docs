# 167 - Current Overload Contactor Breaker

Monitor AC or DC load current with an ACS712 hall-effect current sensor on ADC0/GP26. When current exceeds a configurable trip threshold, open a relay on GPIO 15 to disconnect the load and sound a buzzer alarm on GPIO 14 — functioning as a software-defined circuit breaker.

## Goal
Learn how to use the ACS712 analog current sensor, convert its output voltage to amperes, compare it against a trip threshold with hysteresis, control a relay contactor, and produce an audible alarm — implementing a functional over-current protection device.

## What You Will Build
The ACS712 5A module measures current flowing through the load path. Its VIOUT pin connects to ADC0/GP26. The ARIES board reads this voltage, converts it to amperes using the ACS712 transfer function, and compares it to the trip threshold (5 A). If current exceeds the threshold, the relay on GPIO 15 opens (disconnecting the load), and the buzzer on GPIO 14 sounds until a manual reset button on GPIO 16 is pressed, which resets the system and reconnects the load.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| ACS712 5A Current Sensor Module | `acs712` | Yes | Yes |
| Relay Module (5 V, active-HIGH) | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Push Button (reset) | `button` | Yes | Yes |
| 10 kΩ Pull-up Resistor (button) | `resistor` | No | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ACS712 Module | VCC | 5V | Red | Module power supply |
| ACS712 Module | GND | GND | Black | Common ground |
| ACS712 Module | VIOUT | ADC0 / GP26 | Yellow | Analog current output |
| ACS712 Module | IP+ | Load supply + | Brown | Load current in |
| ACS712 Module | IP− | Load + terminal | Orange | Load current out |
| Relay Module | VCC | 5V | Red | Relay coil supply |
| Relay Module | GND | GND | Black | Common ground |
| Relay Module | IN | GPIO 15 | Green | Relay control from ARIES |
| Relay Module | COM | Load supply | Blue | Main supply to contactor |
| Relay Module | NO | ACS712 IP+ | Blue | Normally-open: connects load when closed |
| Active Buzzer | + | GPIO 14 | Orange | Buzzer positive |
| Active Buzzer | − | GND | Black | Buzzer negative |
| Reset Button | One leg | GPIO 16 | White | Button input |
| Reset Button | Other leg | GND | Black | Active-LOW input |
| Pull-up Resistor | GPIO 16 to 3V3 | — | — | 10 kΩ pull-up for button |

> **Wiring tip:** The ACS712-5B module has a quiescent output of VCC/2 = 2.5 V (with a 5 V supply) when no current flows. Current sensitivity is 185 mV/A for the 5 A variant. Verify the quiescent voltage before connecting a load — use a multimeter to confirm the VIOUT reads ≈ 2.5 V with the load disconnected. If using a 3.3 V-powered ACS712, the quiescent point and sensitivity change — consult the module datasheet and update `ACS_ZERO` accordingly.

## Code
```cpp
// Current Overload Contactor Breaker
// ACS712: ADC0/GP26  |  Relay: GPIO 15  |  Buzzer: GPIO 14  |  Reset: GPIO 16

#define ISENSE_PIN   26   // ADC0/GP26
#define RELAY_PIN    15
#define BUZZER_PIN   14
#define RESET_PIN    16

const float VREF         = 3.3f;
const float ADC_MAX      = 4095.0f;
const float ACS_ZERO     = 2.5f;     // Quiescent output at 0 A (5V ACS712)
const float ACS_MV_A     = 0.185f;   // Sensitivity: 185 mV per Ampere (5A version)
const float TRIP_CURRENT = 5.0f;     // Trip threshold in Amperes
const float RESET_CURRENT = 4.0f;    // Auto-reset only if current is below this

// State machine: 0 = NORMAL, 1 = TRIPPED
int   breakerState  = 0;
float currentAmps   = 0.0f;
float vAcs          = 0.0f;
int   resetBtnState = HIGH;
long  sampleCount   = 0L;

void setup() {
  Serial.begin(115200);
  pinMode(RELAY_PIN,  OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RESET_PIN,  INPUT_PULLUP);
  pinMode(LED_R,      OUTPUT);
  pinMode(LED_G,      OUTPUT);

  // Close relay (load connected) at startup
  digitalWrite(RELAY_PIN,  HIGH);
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(LED_G, HIGH);
  digitalWrite(LED_R, LOW);

  Serial.println("=== Current Overload Contactor Breaker ===");
  Serial.print("Trip threshold: ");
  Serial.print(TRIP_CURRENT, 1);
  Serial.println(" A");
  Serial.println("State: NORMAL — Load connected.");
}

void loop() {
  delay(100);
  sampleCount++;

  // Read ACS712 and convert to current
  int rawI = analogRead(ISENSE_PIN);
  vAcs     = (rawI / ADC_MAX) * VREF;
  currentAmps = (vAcs - ACS_ZERO) / ACS_MV_A;
  if (currentAmps < 0.0f) currentAmps = -currentAmps;  // Absolute value (bidirectional)

  // --- STATE 0: NORMAL OPERATION ---
  if (breakerState == 0) {
    // Log every 5 samples (~500 ms)
    if (sampleCount % 5 == 0) {
      Serial.print("I=");
      Serial.print(currentAmps, 2);
      Serial.println(" A | State: NORMAL");
    }

    if (currentAmps >= TRIP_CURRENT) {
      // TRIP: open relay, sound buzzer, set red LED
      breakerState = 1;
      digitalWrite(RELAY_PIN,  LOW);   // Open contactor — disconnect load
      digitalWrite(BUZZER_PIN, HIGH);  // Sound alarm
      digitalWrite(LED_G, LOW);
      digitalWrite(LED_R, HIGH);
      Serial.print("!!! OVERCURRENT TRIP: ");
      Serial.print(currentAmps, 2);
      Serial.println(" A — Load disconnected. Press RESET button.");
    }
    return;
  }

  // --- STATE 1: TRIPPED ---
  if (breakerState == 1) {
    // Check reset button (active-LOW)
    resetBtnState = digitalRead(RESET_PIN);
    if (resetBtnState == LOW) {
      // Re-read current — only reset if fault has cleared
      int rawI2   = analogRead(ISENSE_PIN);
      float vAcs2 = (rawI2 / ADC_MAX) * VREF;
      float iNow  = (vAcs2 - ACS_ZERO) / ACS_MV_A;
      if (iNow < 0.0f) iNow = -iNow;

      if (iNow < RESET_CURRENT) {
        breakerState = 0;
        digitalWrite(RELAY_PIN,  HIGH);  // Close contactor — reconnect load
        digitalWrite(BUZZER_PIN, LOW);   // Silence alarm
        digitalWrite(LED_G, HIGH);
        digitalWrite(LED_R, LOW);
        sampleCount = 0L;
        Serial.println("Breaker reset — State: NORMAL. Load reconnected.");
      } else {
        Serial.print("Reset blocked — current still high: ");
        Serial.print(iNow, 2);
        Serial.println(" A");
      }
    }
    return;
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Drag a **Potentiometer** component; connect its wiper to **ADC0/GP26** to simulate the ACS712 VIOUT signal.
3. Drag a **Relay** component; connect **IN** to **GPIO 15**, **VCC** to **5V**, **GND** to **GND**.
4. Drag a **Buzzer** component; connect to **GPIO 14** and **GND**.
5. Drag a **Button** component; connect to **GPIO 16** (the other leg connects to **GND**).
6. Paste the code into the editor.
7. Select **Interpreted Mode** from the simulation dropdown.
8. Click **Run**.
9. Rotate the potentiometer upward past mid-scale to simulate a large current (above the ACS712 zero point that maps to 5 A); the relay should open and the buzzer should activate.
10. Press the **Reset** button widget with the potentiometer at a low value; the breaker should reset and reconnect the relay.

## Expected Output
Serial Monitor:
```
=== Current Overload Contactor Breaker ===
Trip threshold: 5.0 A
State: NORMAL — Load connected.
I=1.23 A | State: NORMAL
I=2.47 A | State: NORMAL
I=4.98 A | State: NORMAL
!!! OVERCURRENT TRIP: 5.13 A — Load disconnected. Press RESET button.
Breaker reset — State: NORMAL. Load reconnected.
```

## Expected Canvas Behavior
* The relay widget shows closed (energised) at startup.
* As the potentiometer slider increases, the current reading in the Serial Monitor rises.
* When the simulated current exceeds 5.0 A, the relay opens, the buzzer activates, and the red LED lights.
* Pressing the Reset button widget (with low simulated current) closes the relay and silences the buzzer.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `ACS_ZERO = 2.5f` | Quiescent VIOUT voltage at 0 A; the midpoint of the ACS712 output range. |
| `ACS_MV_A = 0.185f` | Current sensitivity in V/A (185 mV/A for the 5 A variant). |
| `TRIP_CURRENT = 5.0f` | Trip threshold; relay opens and buzzer sounds when current exceeds this. |
| `currentAmps = (vAcs - ACS_ZERO) / ACS_MV_A` | Converts VIOUT voltage to signed current using the ACS712 transfer function. |
| `if (currentAmps < 0.0f) currentAmps = -currentAmps` | Takes the absolute value to handle bidirectional current flow. |
| `digitalWrite(RELAY_PIN, LOW)` | Opens the relay contactor, disconnecting the load from the supply. |
| `resetBtnState = digitalRead(RESET_PIN)` | Reads the reset button; LOW = pressed (active-LOW with pull-up). |
| `if (iNow < RESET_CURRENT)` | Prevents reset while fault persists; only allows reset when current is safely low. |

## Hardware & Safety Concept
* **ACS712 Hall-Effect Sensing**: The ACS712 detects the magnetic field generated by the primary current path using an internal Hall-effect element. The output is electrically isolated from the current path, protecting the MCU from high-voltage transients in the load circuit. This makes it safe to measure mains or high-DC currents without galvanic connection to the ARIES board.
* **Contactor vs Circuit Breaker**: A hardware circuit breaker uses a bimetallic strip or magnetic trip mechanism and operates purely mechanically in microseconds. This project implements a software-defined breaker that responds in ~100 ms (one sample interval). For protecting high-energy circuits from short circuits, a hardware breaker must be used upstream; this project is suitable for over-load protection (sustained excess current, not fault current).
* **Safe Reset Logic**: The breaker prevents reconnection if the fault current is still present. This is an important safety feature: a breaker that resets into a live short circuit will immediately trip again and may damage the relay contacts. Checking current before reset ensures the load fault has been resolved.

## Try This! (Challenges)
1. **Adjustable Trip Threshold**: Read ADC1/GP27 (a potentiometer) and map its 0–4095 range to a trip threshold of 1–10 A. Print the new threshold each time it changes, allowing on-the-fly configuration without reflashing.
2. **Trip Log Counter**: Add a `long tripCount` variable that increments each time the breaker trips. Print the total trip count alongside the "OVERCURRENT TRIP" message, providing a running fault history.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Current reads near -13 A with no load | ACS_ZERO incorrect for module supply voltage | Measure VIOUT with a multimeter at 0 A load; update `ACS_ZERO` to the measured value. |
| Breaker trips immediately at startup | ACS712 quiescent voltage offset or ADC noise | Add a 10 ms settling delay after power-on before reading; check `ACS_ZERO` value. |
| Reset button has no effect | GPIO 16 not pulled up or button wiring incorrect | Confirm `INPUT_PULLUP` on RESET_PIN; verify button connects GPIO 16 to GND when pressed. |
| Buzzer does not sound | Buzzer + and − reversed, or pin mismatch | Confirm BUZZER_PIN = 14; swap buzzer leads if using a passive buzzer module. |
| Relay never opens | RELAY_PIN not defined as OUTPUT | Verify `pinMode(RELAY_PIN, OUTPUT)` is in `setup()`. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [159 - Automatic Battery Capacity Tester](159-automatic-battery-capacity-tester.md)
- [166 - Smart Fan with Temperature Hysteresis](166-smart-fan.md)
- [162 - Solar Tracker Efficiency Logger](162-solar-tracker-efficiency-logger.md)
