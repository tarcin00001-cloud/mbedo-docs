# 132 - Pico BT Environment Monitor HC-05 DHT22 MQ-2 LCD

Build a Bluetooth environment monitoring station that responds to phone requests, reads DHT22 temperature/humidity and MQ-2 gas concentration, and transmits readings back to the smartphone while displaying all values on an I2C LCD.

## Goal
Learn how to build a request-response Bluetooth sensor hub: the HC-05 receives a query command from a smartphone, the Pico reads the appropriate sensor and transmits the formatted result back, while the LCD shows a live dashboard in MicroPython.

## What You Will Build
A Bluetooth environment monitor:
- **HC-05 Bluetooth Module (GP0, GP1)**: Receives sensor query commands and sends readings back.
- **DHT22 Sensor (GP16)**: Reads temperature and humidity on demand.
- **MQ-2 Gas Sensor (GP26)**: Reads gas concentration on demand.
- **I2C 16x2 LCD (GP4, GP5)**: Displays a live dashboard of all readings.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| MQ-2 Gas Sensor | `potentiometer` | Yes (analog input) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 | TXD | GP1 (UART0 RX) | Yellow | BT data to Pico |
| HC-05 | RXD | GP0 (UART0 TX) | Orange | Pico data to BT module |
| HC-05 | VCC | 5V (VBUS) | Red | BT module power |
| HC-05 | GND | GND | Black | Ground reference |
| DHT22 | VCC (+) | 3.3V (3V3) | Red | Sensor power |
| DHT22 | GND (−) | GND | Black | Ground reference |
| DHT22 | DATA | GP16 | Yellow | 1-Wire data signal |
| MQ-2 Gas Sensor | VCC | 5V (VBUS) | Red | Heater power |
| MQ-2 Gas Sensor | GND | GND | Black | Ground reference |
| MQ-2 Gas Sensor | AOUT | GP26 | Yellow | Analog gas concentration |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect HC-05 TXD → GP1 and RXD → GP0. Connect DHT22 DATA to GP16 and MQ-2 AOUT to GP26. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, ADC, UART, I2C
import utime
import dht
from machine_lcd import I2cLcd

# HC-05 on UART0 (GP0 TX, GP1 RX)
bt = UART(0, baudrate=9600, tx=Pin(0), rx=Pin(1))

# DHT22 on GP16
dht_sensor = dht.DHT22(Pin(16))

# MQ-2 on GP26
mq2 = ADC(26)

# I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Cached readings
temp_c = 0.0
humid  = 0.0
gas_pct = 0

# BT command reference:
#   T → send temperature
#   H → send humidity
#   G → send gas level
#   A → send all readings
#   ? → print help

def read_dht():
    global temp_c, humid
    try:
        dht_sensor.measure()
        temp_c = dht_sensor.temperature()
        humid  = dht_sensor.humidity()
    except OSError:
        pass

def read_gas():
    global gas_pct
    raw     = mq2.read_u16()
    gas_pct = min(100, int(raw * 100 / 65535))

def update_lcd():
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr("T:{:.1f}C H:{:.0f}%".format(temp_c, humid))
    lcd.move_to(0, 1)
    lcd.putstr("Gas:{:3d}%  Ready ".format(gas_pct))

# Periodic sensor refresh interval
REFRESH_MS = 2000
last_refresh = utime.ticks_ms() - REFRESH_MS

lcd.clear()
lcd.putstr("BT Env Monitor")
utime.sleep(2)

print("BT Environment Monitor ready.")
print("Commands: T=temp H=humid G=gas A=all ?=help")
bt.write("BT Env Monitor ready. T/H/G/A/?\r\n")

while True:
    now = utime.ticks_ms()

    # Periodic sensor auto-refresh
    if utime.ticks_diff(now, last_refresh) >= REFRESH_MS:
        read_dht()
        read_gas()
        update_lcd()
        last_refresh = now

    # Process incoming BT command
    if bt.any():
        cmd = bt.read(1).decode('utf-8', 'ignore').strip().upper()
        print("BT CMD:", cmd)

        if cmd == 'T':
            msg = "Temp: {:.1f} C\r\n".format(temp_c)
        elif cmd == 'H':
            msg = "Humid: {:.1f} %\r\n".format(humid)
        elif cmd == 'G':
            msg = "Gas: {} %\r\n".format(gas_pct)
        elif cmd == 'A':
            msg = "T:{:.1f}C H:{:.0f}% G:{}%\r\n".format(temp_c, humid, gas_pct)
        elif cmd == '?':
            msg = "T=temp H=humid G=gas A=all\r\n"
        else:
            msg = "Unknown: {}\r\n".format(cmd)

        bt.write(msg)
        print("BT >>", msg.strip())

    utime.sleep_ms(50)
```

## What to Click in MbedO
1. Drag **Pico**, **HC-05**, **DHT22**, **Potentiometer** (MQ-2), and **I2C LCD** onto the canvas.
2. Connect HC-05 to GP0/GP1, DHT22 to GP16, MQ-2 to GP26, and LCD to GP4/GP5. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Open a Bluetooth Serial app on your phone, pair with HC-05, and send `A` to receive all readings. Adjust sensor sliders and query again.

## Expected Output
```
BT Environment Monitor ready.
Commands: T=temp H=humid G=gas A=all ?=help
BT CMD: A
BT >> T:24.5C H:58% G:12%
BT CMD: G
BT >> Gas: 12 %
```
(On screen: "T:24.5C H:58%" and "Gas: 12%  Ready".)

## Expected Canvas Behavior
- The LCD refreshes every 2 seconds with live sensor values. Sending BT commands from a phone app returns formatted sensor data as serial responses.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `utime.ticks_diff(now, last_refresh) >= REFRESH_MS` | Non-blocking 2-second sensor refresh without using `sleep()`, allowing BT commands to be handled at any time. |
| `bt.write(msg)` | Transmits the formatted sensor response string back to the connected Bluetooth phone app. |

## Hardware & Safety Concept: Request-Response Sensor Protocols
This monitor uses a **request-response pattern**: the host (smartphone) issues a query, and the device responds. This is more efficient than continuous streaming because the radio is only active during the exchange, reducing power consumption — an important consideration for battery-powered remote sensors.

## Try This! (Challenges)
1. **Threshold Alerts**: Add a check after every refresh: if gas exceeds 80%, automatically transmit an alert string to the phone without waiting for a query.
2. **CSV Logging**: Format the `A` response as CSV (`temp,humid,gas`) for easy logging in a spreadsheet app.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Phone receives garbled characters | Baud rate mismatch | Ensure the Bluetooth app is set to 9600 baud, matching the UART initialization. |
| DHT22 read fails every cycle | Minimum interval not respected | The 2-second `REFRESH_MS` interval satisfies the DHT22 minimum 2-second sample rate. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [128 - Pico Bluetooth Home Controller HC-05 Relay LCD](128-pico-bluetooth-home-controller-hc05-relay-lcd.md)
- [120 - Pico Multi-Sensor Alarm System LCD](../intermediate/120-pico-multi-sensor-alarm-system-lcd.md)
