# 88 - Pico RFID Access Control

Build a smart door lock access control system that scans RFID cards, verifies their UID, steers a servo to unlock a door, and gives audio/visual status feedback.

## Goal
Learn how to configure the SPI bus, load external RFID libraries, scan cards, verify access permissions, and control servo steering and buzzer warning notes in MicroPython.

## What You Will Build
An automated RFID security lock:
- **MFRC522 RFID Reader**: Scans keycards.
- **Servo Motor (GP10)**: Steers to 90 degrees (unlocked) for 3 seconds on authorized scan, then returns to 0 degrees (locked).
- **LED Green (GP15)**: Turns ON during authorized access.
- **LED Red (GP14)**: Flashes and sounds a warning buzzer if an unauthorized card is scanned.
- **Passive Buzzer (GP13)**: Emits a high chime for success, and a low warning beep for denied access.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes (standard SG90) |
| Red, Green LEDs | `led` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes (passive required) |
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
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The MFRC522 reader requires 3.3V. Connect the servo power to 5V VBUS. All ground lines are connected to the shared ground rail. Connect the LEDs and passive buzzer to their respective GPIO pins.

## Code
```python
from machine import Pin, SPI, PWM
import utime
from mfrc522 import MFRC522

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
        # Failure chime (Low buzzer sound)
        buzzer.freq(220)
        buzzer.duty_u16(32768)
        utime.sleep_ms(400)
        buzzer.duty_u16(0)

print("Access control system armed. Scan keycard...")

while True:
    (status, tag_type) = reader.request(reader.REQIDL)
    
    if status == reader.OK:
        (status, uid) = reader.anticoll()
        
        if status == reader.OK:
            print("UID Scanned:", [hex(x) for x in uid])
            
            # Check permissions
            if uid[0] == ADMIN_UID[0] and uid[1] == ADMIN_UID[1]:
                print(">> ACCESS GRANTED. Unlocking...")
                led_green.value(1)
                sound_chime(True)
                set_lock(True) # Unlock door
                
                utime.sleep(3) # Hold open for 3 seconds
                
                set_lock(False) # Lock door
                led_green.value(0)
                print(">> Door locked.")
            else:
                print(">> ACCESS DENIED! Threat alert.")
                led_red.value(1)
                sound_chime(False)
                
                # Flash red LED warning
                for _ in range(4):
                    led_red.value(1)
                    utime.sleep_ms(100)
                    led_red.value(0)
                    utime.sleep_ms(100)
                    
            utime.sleep(1) # Scan cooldown
            
    utime.sleep_ms(50) # Polling update speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MFRC522 Reader**, **Servo Motor**, **two LEDs**, and **Passive Buzzer** onto the canvas.
2. Connect SPI to **GP16-GP19**, RST to **GP22**, Servo to **GP10**, LEDs to **GP14/GP15**, and Buzzer to **GP13**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the RFID card component to simulate scans.

## Expected Output
```
Access control system armed. Scan keycard...
UID Scanned: ['0xde', '0x12', '0xa4', '0xc3']
>> ACCESS GRANTED. Unlocking...
>> Door locked.
```

## Expected Canvas Behavior
- The green LED lights up and the servo motor rotates to 90 degrees when the authorized card component is clicked.
- Clicking an unauthorized card flashes the red LED and sounds the warning chime.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `uid[0] == ADMIN_UID[0]` | Compares the first bytes of the scanned UID with the authorized admin list. |
| `set_lock(True)` | Sends the duty pulse to steer the servo to the unlocked position (90 degrees). |
| `sound_chime(True)` | Plays a high-pitched double beep to signal successful access. |

## Hardware & Safety Concept: Fail-Secure Lock Design
Automated locks use **fail-secure** designs: if power is lost (e.g. power outage or cut wires), the lock mechanism defaults to the locked state. To ensure safety, automated doors must include a mechanical handle override on the inside (exit side) to allow occupants to escape manually during emergencies, regardless of the system's power state.

## Try This! (Challenges)
1. **Passcode Override**: Add a keypad on GP2-GP9 and allow unlocking the door using either an RFID card or a correct PIN code.
2. **Access Log Datalogger**: Save a log of scanned card UIDs and timestamps to a flash file (`log.csv`).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not rotate | Power supply overload | Connect the servo's power (+) pin to the 5V VBUS pin, not 3.3V. |
| Reading errors on SPI scan | SPI wiring mismatch | Verify that SCK, MOSI, and MISO are connected in the correct order. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [59 - Pico Servo Knob](59-pico-servo-knob.md)
- [63 - Pico RFID Serial](63-pico-rfid-serial.md)
- [64 - Pico Keypad Serial](64-pico-keypad-serial.md)
