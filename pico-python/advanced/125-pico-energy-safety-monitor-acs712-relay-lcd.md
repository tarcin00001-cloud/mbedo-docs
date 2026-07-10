# 125 - Pico Energy Safety Monitor ACS712 Relay LCD

Build a power-line energy safety monitor that reads an ACS712 current sensor, calculates real-time power consumption, trips a protection relay if current exceeds a safe limit, and displays live readings and status on an I2C LCD.

## Goal
Learn how to implement over-current protection logic using an ACS712 current sensor, control a protection relay as a circuit breaker, trigger a fault buzzer, and display live current, power, and status on an I2C LCD in MicroPython.

## What You Will Build
An energy safety monitor with circuit breaker:
- **ACS712 Current Sensor (GP26)**: Reads line current in amps.
- **Relay — Protection Breaker (GP10)**: Normally-closed (NC) configuration: trips open on over-current.
- **Red LED (GP13)**: ON during fault condition.
- **Buzzer (GP15)**: Sounds on over-current trip.
- **Reset Button (GP12)**: Resets the trip and re-closes the relay after the fault is cleared.
- **I2C 16x2 LCD (GP4, GP5)**: Displays current, calculated power, and circuit status.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| ACS712 Current Sensor (5A or 20A) | `potentiometer` | Yes (represented by slider) | Yes |
| Relay Module (Protection Breaker) | `relay` | Yes | Yes |
| Red LED | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button (Reset) | `button` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ACS712 Module | VCC (+) | 5V (VBUS) | Red | Sensor heater power (5V) |
| ACS712 Module | GND (−) | GND | Black | Ground reference |
| ACS712 Module | OUT (Analog) | GP26 | Yellow | Analog current signal |
| Relay Module | IN | GP10 | Orange | Protection relay control signal |
| Red LED | Anode (+, via 330 Ω) | GP13 | Red | Fault indicator |
| Red LED | Cathode (−) | GND | Black | Shared return path to ground |
| Active Buzzer | VCC (+) | GP15 | Red | Fault alarm output |
| Reset Button | Terminal 1 | GP12 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP12 to GND |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the ACS712 output to GP26. Connect the protection relay to GP10. Connect the fault LED to GP13, buzzer to GP15, and reset button to GP12. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, ADC, I2C
import utime
from machine_lcd import I2cLcd

acs712     = ADC(26)  # GP26 = ADC channel 0
relay      = Pin(10, Pin.OUT)
fault_led  = Pin(13, Pin.OUT)
buzzer     = Pin(15, Pin.OUT)
btn_reset  = Pin(12, Pin.IN, Pin.PULL_UP)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# ACS712-5A sensitivity: 185 mV/A
SENSITIVITY_MV_A = 185.0
ZERO_RAW         = 32767  # Zero-current ADC mid-point
SAMPLES          = 50     # Oversampling count

# Protection limits
CURRENT_TRIP_A  = 3.5    # Over-current trip threshold (amps)
NOMINAL_VOLTS   = 230.0  # Nominal supply voltage for power calculation

# States
CLOSED   = "CLOSED"   # Relay energized, circuit connected
TRIPPED  = "TRIPPED"  # Over-current protection relay open

state = CLOSED
relay.value(1)  # Relay normally-closed: value 1 = circuit connected
fault_led.value(0)
buzzer.value(0)

def read_current():
    total = sum(acs712.read_u16() for _ in range(SAMPLES))
    avg   = total / SAMPLES
    v_now = avg * 3.3 / 65535
    v_zero = ZERO_RAW * 3.3 / 65535
    return abs((v_now - v_zero) * 1000.0 / SENSITIVITY_MV_A)

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Energy Monitor")
lcd.move_to(0, 1)
lcd.putstr("Calibrating...")
utime.sleep(1)

print("Energy safety monitor active.")

while True:
    current_a = read_current()
    power_w   = current_a * NOMINAL_VOLTS
    
    # --- Over-current protection ---
    if state == CLOSED and current_a > CURRENT_TRIP_A:
        state = TRIPPED
        relay.value(0)      # Open relay — disconnect load
        fault_led.value(1)
        buzzer.value(1)
        print(">> OVER-CURRENT TRIP! {:.2f} A".format(current_a))
        
    # --- Manual reset ---
    if btn_reset.value() == 0 and state == TRIPPED:
        if current_a < CURRENT_TRIP_A:  # Only reset if current is safe
            state = CLOSED
            relay.value(1)  # Re-close relay
            fault_led.value(0)
            buzzer.value(0)
            print(">> Circuit reset. State: CLOSED.")
        else:
            print(">> Reset rejected: current still high ({:.2f} A)".format(current_a))
        utime.sleep_ms(500)
        
    # --- LCD Update ---
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("I:{:.2f}A P:{:.0f}W".format(current_a, power_w))
    lcd.move_to(0, 1)
    if state == TRIPPED:
        lcd.putstr("!! TRIP !! Reset?")
    else:
        lcd.putstr("Status:  NORMAL")
        
    # --- Alarm pulsing in tripped state ---
    if state == TRIPPED:
        buzzer.value(1)
        utime.sleep_ms(200)
        buzzer.value(0)
        utime.sleep_ms(200)
    else:
        utime.sleep_ms(300)
        
    print("I:{:.2f}A P:{:.0f}W Status:{}".format(current_a, power_w, state))
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer** (ACS712), **Relay**, **Red LED**, **Active Buzzer**, **Push Button** (Reset), and **I2C LCD** onto the canvas.
2. Connect Potentiometer to **GP26**, Relay to **GP10**, LED to **GP13**, Buzzer to **GP15**, Button to **GP12**, and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the potentiometer above the trip threshold to trigger the over-current protection. Click Reset to re-close the circuit.

## Expected Output
```
Energy safety monitor active.
I:1.50A P:345W Status:CLOSED
>> OVER-CURRENT TRIP! 4.20 A
I:4.20A P:966W Status:TRIPPED
>> Circuit reset. State: CLOSED.
```

## Expected Canvas Behavior
- The relay component opens (circuit disconnect) when the potentiometer slider pushes the simulated current above 3.5A. The fault LED and buzzer activate. The LCD shows "!! TRIP !!". Clicking the Reset button while the slider is below the trip threshold re-closes the relay.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `relay.value(0)` | Opens the normally-closed relay to disconnect the load circuit on over-current detection. |
| `if current_a < CURRENT_TRIP_A` | Safety interlock: the reset is rejected if the fault current is still present, preventing immediate re-trip. |

## Hardware & Safety Concept: Electronic Circuit Breakers
A physical circuit breaker trips on over-current by thermally bending a bimetal strip or using an electromagnetic solenoid to latch the contact open. This circuit is an **electronic solid-state equivalent**: the microcontroller acts as the trip logic, and the relay acts as the contact. Unlike a physical breaker, software trip conditions can be programmed precisely and changed without hardware modifications.

## Try This! (Challenges)
1. **Time-Delay Trip**: Add a 100 ms timer so brief current spikes (motor startup) do not cause a false trip.
2. **Energy Logger**: Accumulate watt-hours over time using `power_w * elapsed_time / 3600000` and display total kWh on the LCD.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay trips immediately on startup | Zero-current calibration off | Measure the raw ADC value with no load connected and set `ZERO_RAW` to that value. |
| Reset button does not restore circuit | Current still above threshold | Lower the potentiometer slider first, then press reset. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [112 - Pico ACS712 Current Monitor LCD](112-pico-acs712-current-monitor-lcd.md)
- [118 - Pico Dual Relay Timer Controller LCD](118-pico-dual-relay-timer-controller-lcd.md)
- [120 - Pico Multi-Sensor Alarm System LCD](120-pico-multi-sensor-alarm-system-lcd.md)
