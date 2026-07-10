# 96 - Pico RFID Access Control LCD

Build a smart door lock access control console that scans RFID cards, verifies their UID, steers a servo to unlock a door, and displays security layout updates on an I2C character LCD.

## Goal
Learn how to configure the SPI communication bus, scan passive keycards, verify permissions, steer servo motors to physical limits, and print formatting variables on an I2C character LCD in MicroPython.

## What You Will Build
An automated RFID security locking panel:
- **MFRC522 RFID Reader**: Scans keycards.
- **Servo Motor (GP10)**: Steers to 90 degrees (unlocked) for 3 seconds on authorized scan, then returns to 0 degrees (locked).
- **LED Green (GP15)**: Turns ON during authorized access.
- **LED Red (GP14)**: Flashes during unauthorized scans.
- **Passive Buzzer (GP13)**: Emits chimes for access granted/denied.
- **I2C 16x2 LCD (GP4, GP5)**: Displays active lock states ("ACCESS GRANTED" or "ACCESS DENIED") and scanned card UIDs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes (standard SG90) |
| Red, Green LEDs | `led` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes (passive required) |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 330 Ω Resistors × 2 | `resistor` | Optional in MbedO | Yes — one per LED |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | 3.3V / GND | 3.3V / GND | Red / Black | Reader power |
| MFRC522 | MISO / MOSI | GP16 / GP19 | Yellow / Green | SPI data lines |
| MFRC522 | SCK / SDA (CS) | GP18 / GP17 | Blue / Orange | SPI clock and select |
| MFRC522 | RST | GP22 | White | Reset line |
| Servo Motor | Control (PWM) | GP10 | Orange | PWM servo signal |
| Servo Motor | Power (+) | 5V (VBUS) | Red | Servo power supply (5V) |
| LED Red | Anode (+, longer leg) | GP14 | Red | Access Denied indicator |
| LED Green | Anode (+, longer leg) | GP15 | Green | Access Granted indicator |
| Passive Buzzer | Signal (+) | GP13 | Grey | Warning chime signal (PWM) |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the MFRC522 reader VCC to 3.3V, and the servo power pin to 5V VBUS. Connect the LCD to GP4/GP5. All ground lines are connected to the shared ground rail. Connect the LEDs and passive buzzer to their respective pins.

## Code
```python
from machine import Pin, SPI, PWM, I2C
import utime
from mfrc522 import MFRC522
from machine_lcd import I2cLcd

# Initialize SPI 0 on GP16-19 pins
sck_pin  = Pin(18, Pin.OUT)
mosi_pin = Pin(19, Pin.OUT)
miso_pin = Pin(16, Pin.OUT)
sda_pin  = Pin(17, Pin.OUT)
rst_pin  = Pin(22, Pin.OUT)

spi = SPI(0, baudrate=1000000, polarity=0, phase=0,
          sck=sck_pin, mosi=mosi_pin, miso=miso_pin)

reader = MFRC522(spi, sda_pin, rst_pin)

# Set up Servo on GP10 (50 Hz PWM)
servo = PWM(Pin(10))
servo.freq(50)

# Set up Passive Buzzer on GP13
buzzer = PWM(Pin(13))

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

led_red   = Pin(14, Pin.OUT)
led_green = Pin(15, Pin.OUT)

# Servo duty cycle limits
MIN_DUTY = 1638 # Locked (0 deg)
MAX_DUTY = 4915 # Unlocked (90 deg)

# Authorized master card UID prefix
ADMIN_UID = [0xDE, 0x12]

# Ensure start state is locked and silent
servo.duty_u16(MIN_DUTY)
led_red.value(0)
led_green.value(0)
buzzer.duty_u16(0)

def set_lock(open_state):
    duty = MAX_DUTY if open_state else MIN_DUTY
    servo.duty_u16(duty)

def sound_chime(success):
    if success:
        # Success chime (High pitched double beep)
        for freq in [1000, 1500]:
            buzzer.freq(freq)
            buzzer.duty_u16(32768)
            utime.sleep_ms(80)
        buzzer.duty_u16(0)
    else:
        # Failure chime (Low warning tone)
        buzzer.freq(220)
        buzzer.duty_u16(32768)
        utime.sleep_ms(400)
        buzzer.duty_u16(0)

# Initial display setup
lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Security Console")
lcd.move_to(0, 1)
lcd.putstr("Scan Card...")

print("Access control system armed. Scan keycard...")

while True:
    (status, tag_type) = reader.request(reader.REQIDL)
    
    if status == reader.OK:
        (status, uid) = reader.anticoll()
        
        if status == reader.OK:
            uid_str = "".join([hex(x)[2:].upper().zfill(2) for x in uid[:4]])
            print("UID Scanned:", uid_str)
            
            lcd.clear()
            lcd.move_to(0, 0)
            lcd.putstr("ID: " + uid_str)
            
            # Check permissions
            if uid[0] == ADMIN_UID[0] and uid[1] == ADMIN_UID[1]:
                print(">> ACCESS GRANTED. Unlocking...")
                lcd.move_to(0, 1)
                lcd.putstr("ACCESS: GRANTED")
                
                led_green.value(1)
                sound_chime(True)
                set_lock(True) # Unlock door
                
                utime.sleep(3) # Hold open for 3 seconds
                
                set_lock(False) # Lock door
                led_green.value(0)
                print(">> Door locked.")
            else:
                print(">> ACCESS DENIED! Threat alert.")
                lcd.move_to(0, 1)
                lcd.putstr("ACCESS: DENIED")
                
                led_red.value(1)
                sound_chime(False)
                
                # Flash red LED warning
                for _ in range(4):
                    led_red.value(1)
                    utime.sleep_ms(100)
                    led_red.value(0)
                    utime.sleep_ms(100)
                    
            # Reset LCD message
            lcd.clear()
            lcd.move_to(0, 0)
            lcd.putstr("Security Console")
            lcd.move_to(0, 1)
            lcd.putstr("Scan Card...")
            
            utime.sleep(1) # Scan cooldown
            
    utime.sleep_ms(50) # Polling update speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MFRC522 Reader**, **Servo Motor**, **two LEDs**, **Passive Buzzer**, and **I2C LCD** onto the canvas.
2. Connect SPI to **GP16-GP19**, RST to **GP22**, Servo to **GP10**, LEDs to **GP14/GP15**, Buzzer to **GP13**, and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the RFID card component to simulate scans and observe the LCD and servo.

## Expected Output
```
Access control system armed. Scan keycard...
UID Scanned: DE12A4C3
>> ACCESS GRANTED. Unlocking...
>> Door locked.
```
(On screen: "ID: DE12A4C3" and "ACCESS: GRANTED" when scanned, servo unlocking.)

## Expected Canvas Behavior
- The green LED lights up, the servo motor rotates to 90 degrees, and the LCD screen displays "ACCESS: GRANTED" when the authorized card component is clicked.
- Clicking an unauthorized card flashes the red LED, displays "ACCESS: DENIED", and sounds the warning chime.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `uid_str = "".join(...)` | Formats the scanned byte array into a clean hexadecimal string. |
| `lcd.putstr("ACCESS: GRANTED")` | Prints the approval status on the second line of the display. |
| `set_lock(True)` | Unlocks the door latch by steering the servo to 90 degrees. |

## Hardware & Safety Concept: Decoupling Capacitors
When servo motors start rotating, they draw high current, which can cause electrical noise on the power rails. This noise can interfere with sensitive I2C displays, causing them to show random blocks or freeze. Adding a **100 µF electrolytic capacitor** across the power and ground rails near the servo acts as a reservoir, filtering out voltage drops and protecting other components.

## Try This! (Challenges)
1. **Passcode Override**: Add a keypad on GP2-GP9 and allow unlocking the door using either an RFID card or a correct PIN code.
2. **Access Log Datalogger**: Save a log of scanned card UIDs and timestamps to a flash file (`log.csv`).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD displays random blocks or freezes | Electrical noise from servo | Add a decoupling capacitor across the power rails, or power the servo from a separate supply. |
| Servo does not rotate | Power rail issue | Connect the servo's power (+) pin to the 5V VBUS pin, not 3.3V. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [59 - Pico Servo Knob](59-pico-servo-knob.md)
- [63 - Pico RFID Serial](63-pico-rfid-serial.md)
- [88 - Pico RFID Access Control](88-pico-rfid-access-control.md)
- [94 - Pico RFID LCD Status](94-pico-rfid-lcd-status.md)
