# 173 - Pico Asyncio Button Debouncer

Build an advanced button press analyzer that registers hardware interrupts (IRQs), debounces them asynchronously using sleep checks, and displays raw contact bounces vs. clean registered presses on an OLED.

## Goal
Learn how to combine high-speed hardware interrupts (IRQs) with cooperative multitasking (`uasyncio`) to filter out mechanical button contact bounce without locking the main thread.

## What You Will Build
An interrupt-driven debounce visualizer:
- **Push Button (GP14)**: Triggers hardware interrupts on contact.
- **Red LED (GP15)**: Toggles on clean debounced button presses.
- **Active Buzzer (GP16)**: Chirps briefly on a valid press.
- **SSD1306 OLED (GP4, GP5)**: Displays a real-time counter showing "Raw Triggers" vs. "Clean Presses".

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| SSD1306 OLED Display | `oled` | Yes | Yes |
| Green / Red LED | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Button | Terminal 1 | GP14 | White | Interrupt input pin |
| Button | Terminal 2 | GND | Black | Ground return |
| Red LED | Anode (via 330 Ω) | GP15 | Red | Clean press indicator |
| Red LED | Cathode | GND | Black | Ground return |
| Buzzer | VCC (+) | GP16 | Blue | Valid chirp output |
| Buzzer | GND | GND | Black | Ground return |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The button on GP14 will generate mechanical noise (bounce) when pressed. The OLED connects to GP4/GP5. The LED on GP15 and buzzer on GP16 act as feedback actuators.

## Code
```python
import uasyncio as asyncio
from machine import Pin, I2C
import utime, ssd1306

button_pin = Pin(14, Pin.IN, Pin.PULL_UP)
led = Pin(15, Pin.OUT)
buzzer = Pin(16, Pin.OUT)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# Global variables shared with ISR
raw_triggers = 0
clean_presses = 0
irq_fired = False

def button_isr(pin):
    """Hardware interrupt handler. Executes immediately on falling edge."""
    global raw_triggers, irq_fired
    raw_triggers += 1
    irq_fired = True

# Attach interrupt to button pin
button_pin.irq(trigger=Pin.IRQ_FALLING, handler=button_isr)

async def debounce_task():
    """Asynchronous task that waits for interrupts and debounces them."""
    global irq_fired, clean_presses
    print("Debounce analyzer task active.")
    while True:
        if irq_fired:
            # Clear flag immediately
            irq_fired = False
            
            # Wait 50 ms asynchronously to let contacts settle
            await asyncio.sleep_ms(50)
            
            # Re-read the pin: if still LOW, it is a valid press
            if button_pin.value() == 0:
                clean_presses += 1
                led.toggle()
                
                # Active buzzer chirp
                buzzer.value(1)
                await asyncio.sleep_ms(30)
                buzzer.value(0)
                
                # Wait until the button is fully released before accepting new presses
                while button_pin.value() == 0:
                    await asyncio.sleep_ms(20)
        # Yield to let other tasks run
        await asyncio.sleep_ms(5)

async def display_task():
    """Updates the OLED screen with statistics."""
    while True:
        oled.fill(0)
        oled.rect(0, 0, 128, 14, 1)
        oled.text("BOUNCE ANALYZER", 4, 3, 0)
        
        oled.text("Raw IRQs : {}".format(raw_triggers), 10, 24, 1)
        oled.text("Clean Px : {}".format(clean_presses), 10, 38, 1)
        
        # Calculate bounce ratio if clean presses > 0
        if clean_presses > 0:
            ratio = raw_triggers / clean_presses
            oled.text("Ratio    : {:.1f}:1".format(ratio), 10, 52, 1)
        else:
            oled.text("Ratio    : 0.0:1", 10, 52, 1)
            
        oled.rect(0, 0, 128, 64, 1)
        oled.show()
        await asyncio.sleep_ms(100)

async def main():
    t1 = asyncio.create_task(debounce_task())
    t2 = asyncio.create_task(display_task())
    await asyncio.gather(t1, t2)

# Start event loop
asyncio.run(main())
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button**, **OLED**, **LED**, and **Buzzer** onto the canvas.
2. Connect Button to **GP14**, LED to **GP15**, Buzzer to **GP16**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Press the Button multiple times. In a physical switch, raw IRQ counts will exceed clean presses due to contact bounce. Verify that the debounce loop filters them correctly.

## Expected Output
Terminal:
```
Debounce analyzer task active.
```
(On OLED: `Raw IRQs: 4` / `Clean Px: 3` showing the bounce filtering.)

## Expected Canvas Behavior
* Pressing the button immediately updates the raw count and toggles the LED.
* The buzzer emits a short, clean beep for each valid click.
* The OLED shows the filtering statistics.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `button_pin.irq(...)` | Configures GP14 to trigger the `button_isr` function on a high-to-low voltage transition. |
| `await asyncio.sleep_ms(50)` | Yields control back to the event loop for 50 milliseconds while waiting for contact bounce to settle. |

## Hardware & Safety Concept: Switch Contact Bounce
Mechanical switches contain spring-loaded metal contacts. When pressed, these contacts physically bounce against each other for several milliseconds before making solid electrical contact, causing the output voltage to rapidly switch between HIGH and LOW. Un-debounced switches can cause microcontrollers to register a single click as multiple inputs, which can corrupt counters or menu navigation.

## Try This! (Challenges)
1. **Double Click Register**: Write an async task that registers a "Double Click" event if two clean presses occur within 400 ms.
2. **Dynamic Sleep**: Adjust the debounce sleep time dynamically from 10 ms to 100 ms using a potentiometer on GP26, and find the minimum stable setting.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Double trigger on a single press | Debounce delay too short | Increase the `await asyncio.sleep_ms(50)` debounce window to 80 ms or 100 ms. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [16 - Pico Button LED](../../beginner/16-pico-button-led.md)
- [156 - Pico Interrupt-Driven Encoder Tachometer OLED](../advanced/156-pico-interrupt-driven-encoder-tachometer-oled.md)
