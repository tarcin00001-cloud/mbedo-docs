# 94 - Pico RFID LCD Status

Build a smart door lock access control panel that scans RFID cards, verifies their UID, and displays the card's serial ID and permission status on an I2C LCD screen.

## Goal
Learn how to configure the SPI communication bus, load external RFID libraries, scan cards, verify access permissions, and update status text on an I2C character LCD in MicroPython.

## What You Will Build
An RFID access control dashboard:
- **MFRC522 RFID Reader**: Scans keycards.
- **I2C 16x2 LCD (GP4, GP5)**: Displays scanned UIDs and access status ("ACCESS GRANTED" or "ACCESS DENIED").
- **LED Green (GP15)**: Turns ON during authorized access.
- **LED Red (GP14)**: Flashes during unauthorized scans.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Red, Green LEDs | `led` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 330 Ω Resistors × 2 | `resistor` | Optional in MbedO | Yes — one per LED |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | 3.3V / GND | 3.3V / GND | Red / Black | Reader power |
| MFRC522 | MISO / MOSI | GP16 / GP19 | Yellow / Green | SPI data lines |
| MFRC522 | SCK / SDA (CS) | GP18 / GP17 | Blue / Orange | SPI clock and select |
| MFRC522 | RST | GP22 | White | Reset line |
| LED Red | Anode (+, longer leg) | GP14 | Red | Access Denied indicator |
| LED Green | Anode (+, longer leg) | GP15 | Green | Access Granted indicator |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the MFRC522 reader VCC to 3.3V. Connect the LCD to GP4/GP5. All ground lines are connected to the shared ground rail. Connect the LEDs to their respective GPIO pins.

## Code
```python
from machine import Pin, SPI, I2C
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

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

led_red   = Pin(14, Pin.OUT)
led_green = Pin(15, Pin.OUT)

# Authorized master card UID prefix
ADMIN_UID = [0xDE, 0x12]

# Ensure start state is locked and silent
led_red.value(0)
led_green.value(0)

lcd.clear()
lcd.putstr("Security Console")
lcd.move_to(0, 1)
lcd.putstr("Scan Card...")

while True:
    (status, tag_type) = reader.request(reader.REQIDL)
    
    if status == reader.OK:
        (status, uid) = reader.anticoll()
        
        if status == reader.OK:
            # Format UID to string
            uid_str = "".join([hex(x)[2:].upper().zfill(2) for x in uid[:4]])
            print("UID Scanned:", uid_str)
            
            lcd.clear()
            lcd.move_to(0, 0)
            lcd.putstr("ID: " + uid_str)
            
            # Check permissions
            if uid[0] == ADMIN_UID[0] and uid[1] == ADMIN_UID[1]:
                print(">> ACCESS GRANTED.")
                led_green.value(1)
                lcd.move_to(0, 1)
                lcd.putstr("ACCESS: GRANTED")
                utime.sleep(3) # Hold status for 3 seconds
                led_green.value(0)
            else:
                print(">> ACCESS DENIED!")
                led_red.value(1)
                lcd.move_to(0, 1)
                lcd.putstr("ACCESS: DENIED")
                
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
1. Drag the **Raspberry Pi Pico**, **MFRC522 Reader**, **two LEDs**, and **I2C LCD** onto the canvas.
2. Connect SPI to **GP16-GP19**, RST to **GP22**, LEDs to **GP14/GP15**, and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the RFID card component to simulate scans and observe the LCD.

## Expected Output
```
UID Scanned: DE12A4C3
>> ACCESS GRANTED.
```
(On screen: "ID: DE12A4C3" and "ACCESS: GRANTED" when scanned.)

## Expected Canvas Behavior
- The green LED lights up and the LCD screen updates to show "ACCESS: GRANTED" when the authorized card component is clicked.
- Clicking an unauthorized card flashes the red LED and displays "ACCESS: DENIED".

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `uid_str = "".join(...)` | Formats the scanned byte array into a clean hexadecimal string. |
| `lcd.putstr("ACCESS: GRANTED")` | Prints the approval status on the second line of the display. |

## Hardware & Safety Concept: Display Fail-Safe Layouts
When designing access control panels, the status screen must display the card ID alongside the verification result. In commercial buildings, security logs are monitored in real time. Displaying the scanned card ID allows security guards to quickly match the card face with the database log during unauthorized access attempts.

## Try This! (Challenges)
1. **Indicator Buzzer**: Connect a buzzer on GP13 and sound a high double beep on success, and a low warning tone on failure.
2. **Access Counter**: Maintain a counter of successful entries and display it on the LCD screen.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD displays EIO errors on SPI scan | I2C/SPI wiring conflict | Ensure the LCD is wired to GP4/GP5, and MFRC522 SPI pins are wired to GP16-GP19 (SPI0). |
| Cards are not detected | SPI wire disconnected | Check all SPI connections and ensure SCK, MOSI, and MISO are wired correctly. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [63 - Pico RFID Serial](63-pico-rfid-serial.md)
- [88 - Pico RFID Access Control](88-pico-rfid-access-control.md)
