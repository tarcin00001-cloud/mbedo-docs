# 171 - Pico Asyncio LED Blink

Build a non-blocking dual-LED flasher that utilizes MicroPython's cooperative multitasking library `uasyncio` to flash two LEDs at independent rates without using standard blocking delays.

## Goal
Learn how to implement cooperative multitasking using the `uasyncio` library in MicroPython, write asynchronous tasks with `async` and `await`, and coordinate multiple concurrent loops.

## What You Will Build
A concurrent LED flasher:
- **Green LED (GP15)**: Toggles every 500 ms asynchronously.
- **Red LED (GP16)**: Toggles every 1200 ms asynchronously.
- **Push Button (GP14)**: Triggers an interrupt-like check to print the loop status without interrupting the LED timing.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Green LED | `led` | Yes | Yes |
| Red LED | `led` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| 330 Ω Resistors × 2 | `resistor` | Optional | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Green LED | Anode (via 330 Ω) | GP15 | Green | Fast blink indicator |
| Green LED | Cathode | GND | Black | Ground return |
| Red LED | Anode (via 330 Ω) | GP16 | Red | Slow blink indicator |
| Red LED | Cathode | GND | Black | Ground return |
| Button | Terminal 1 | GP14 | White | Status query input |
| Button | Terminal 2 | GND | Black | Button return |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the Green LED to GP15 and the Red LED to GP16, each in series with a 330 Ω resistor. The Button is wired to GP14 with an internal pull-up.

## Code
```python
import uasyncio as asyncio
from machine import Pin
import utime

led_green = Pin(15, Pin.OUT)
led_red = Pin(16, Pin.OUT)
btn = Pin(14, Pin.IN, Pin.PULL_UP)

async def blink_green():
    """Asynchronous task to toggle the green LED every 500ms."""
    while True:
        led_green.toggle()
        # Yield execution to other tasks for 500ms
        await asyncio.sleep_ms(500)

async def blink_red():
    """Asynchronous task to toggle the red LED every 1200ms."""
    while True:
        led_red.toggle()
        # Yield execution to other tasks for 1200ms
        await asyncio.sleep_ms(1200)

async def monitor_button():
    """Asynchronous task to poll the button without blocking the loop."""
    print("Monitor task active. Press button to log state.")
    while True:
        if btn.value() == 0:
            print(">> Button pressed! Current time ms:", utime.ticks_ms())
            # Simple debounce: wait for button release asynchronously
            while btn.value() == 0:
                await asyncio.sleep_ms(20)
        await asyncio.sleep_ms(50)  # Polling interval

async def main():
    # Schedule all three tasks to run concurrently
    task1 = asyncio.create_task(blink_green())
    task2 = asyncio.create_task(blink_red())
    task3 = asyncio.create_task(monitor_button())
    
    # Run forever
    await asyncio.gather(task1, task2, task3)

# Start the uasyncio event loop
print("Starting cooperative multitasking event loop...")
asyncio.run(main())
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two LEDs**, and **Push Button** onto the canvas.
2. Connect Green LED to **GP15**, Red LED to **GP16**, and Button to **GP14**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Click the Button during the simulation and verify in the terminal that the button press registers immediately without causing any stutter in the LED blinking.

## Expected Output
Terminal:
```
Starting cooperative multitasking event loop...
Monitor task active. Press button to log state.
>> Button pressed! Current time ms: 4250
>> Button pressed! Current time ms: 7800
```

## Expected Canvas Behavior
* Green LED (GP15) blinks at a rapid, steady 1 Hz (500 ms ON, 500 ms OFF).
* Red LED (GP16) blinks at a slower rate (1200 ms ON, 1200 ms OFF).
* Button press registers instantly without blocking the LED blinking.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `import uasyncio as asyncio` | Imports the MicroPython cooperative multitasking framework. |
| `await asyncio.sleep_ms(500)` | Suspends the current coroutine, allowing the event loop to run other scheduled tasks. |
| `asyncio.run(main())` | Starts the event loop and executes the passed coroutine, running until it returns. |

## Hardware & Safety Concept: Cooperative vs Preemptive Multitasking
Cooperative multitasking relies on tasks explicitly yielding control (using `await`) to the scheduler so other tasks can execute. If a single task performs a blocking operation (like `utime.sleep(2.0)` or a long calculation), the entire system freezes. Preemptive multitasking (used in desktop OSes) forces tasks to yield, but cooperative multitasking is lightweight, uses less memory, and is highly predictable for microcontrollers.

## Try This! (Challenges)
1. **Third LED Chase**: Add a Blue LED on GP13 that cycles through a breathing pattern using PWM, controlled by a fourth asynchronous task.
2. **Frequency Doubler**: Modify the button monitor task to double the green LED speed for 5 seconds when pressed, returning it to normal speed after.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Blinking freezes entirely | A blocking call was used | Never use `utime.sleep()` or `time.sleep()` in an async task. Always use `await asyncio.sleep()` or `await asyncio.sleep_ms()`. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [04 - Multiple LED Chase](../../beginner/04-multiple-led-chase.md)
- [120 - Pico Multi-Sensor Alarm System LCD](../intermediate/120-pico-multi-sensor-alarm-system-lcd.md)
