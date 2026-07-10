# 182 - Pico Multithreaded Mutex Locks

Build a secure transaction station that uses mutex locks to prevent race conditions when two processor cores access and modify a shared ledger variable simultaneously.

## Goal
Learn how to implement mutual exclusion (mutex locks) using MicroPython's `_thread` module, understand how race conditions occur in multi-core systems, and protect shared variables.

## What You Will Build
A synchronized balance manager:
- **Core 0 (Main Thread)**: Handles manual deposits when a Push Button (GP13) is clicked, updating a shared balance variable.
- **Core 1 (Background Thread)**: Simulates automatic fee deductions every 1.5 seconds, modifying the same shared balance.
- **Ledger Lock (Mutex)**: Prevents both cores from accessing or modifying the balance at the same instant.
- **16x2 I2C LCD (GP4, GP5)**: Displays the current balance and the status of the lock (LOCKED vs. FREE).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Button | Terminal 1 | GP13 | White | Deposit trigger pin |
| Button | Terminal 2 | GND | Black | Ground return |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the deposit button to GP13 with an internal pull-up. The LCD screen uses GP4/GP5.

## Code
```python
import _thread
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

btn_deposit = Pin(13, Pin.IN, Pin.PULL_UP)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Shared resource
balance = 100
# Allocate a hardware mutex lock
ledger_lock = _thread.allocate_lock()

# Status flag for LCD
lock_status = "FREE  "

def core1_thread():
    """Fee deduction task running on Core 1."""
    global balance, lock_status
    print("[Core 1] Background deduction thread active.")
    while True:
        # 1. Acquire the lock before accessing shared variables
        # If Core 0 currently holds the lock, Core 1 blocks here until it is released
        ledger_lock.acquire()
        lock_status = "LOCKED"
        
        # Simulate processing time inside the critical section
        utime.sleep_ms(100)
        
        # Apply automatic fee deduction
        balance -= 5
        print("[Core 1] Deducted 5. Balance is:", balance)
        
        # 2. Release the lock immediately when done
        lock_status = "FREE  "
        ledger_lock.release()
        
        # Wait 1.5 seconds before next fee deduction
        utime.sleep_ms(1500)

# Start background thread on Core 1
_thread.start_new_thread(core1_thread, ())

lcd.clear()
lcd.putstr("Ledger Initialised\nBalance: $100")
utime.sleep(1.0)

print("[Core 0] Deposit interface ready.")

while True:
    # Check for deposit button click (active LOW)
    if btn_deposit.value() == 0:
        # Acquire lock to update the ledger safely
        ledger_lock.acquire()
        lock_status = "LOCKED"
        
        # Add deposit amount
        balance += 20
        print("[Core 0] Deposited 20. Balance is:", balance)
        
        # Debounce button press inside lock to hold resource
        utime.sleep_ms(200)
        while btn_deposit.value() == 0:
            utime.sleep_ms(10)
            
        lock_status = "FREE  "
        ledger_lock.release()
        
    # Update LCD display safely
    # We acquire the lock while reading the balance to ensure it doesn't change mid-read
    ledger_lock.acquire()
    current_balance = balance
    current_status = lock_status
    ledger_lock.release()
    
    lcd.clear()
    lcd.putstr("Balance: ${}\n".format(current_balance))
    lcd.putstr("Mutex: {}".format(current_status))
    
    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Push Button**, and **I2C LCD** onto the canvas.
2. Connect Button to **GP13**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe the balance decreasing by 5 every 1.5 seconds. Click the button to deposit $20. Verify that deposits are added immediately and the lock status changes on the display.

## Expected Output
Terminal:
```
[Core 1] Background deduction thread active.
[Core 0] Deposit interface ready.
[Core 1] Deducted 5. Balance is: 95
[Core 0] Deposited 20. Balance is: 115
[Core 1] Deducted 5. Balance is: 110
```

## Expected Canvas Behavior
* Startup: LCD reads `Balance: $100`.
* Every 1.5 seconds: The balance decreases by $5. Mutex flashes `LOCKED` briefly.
* Click Button (GP13): Balance increases by $20 immediately.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `_thread.allocate_lock()` | Creates a new lock object (mutex) to synchronize access to shared memory. |
| `ledger_lock.acquire()` | Locks the mutex. Blocks execution if another thread currently holds the lock. |
| `ledger_lock.release()` | Unlocks the mutex, permitting waiting threads to acquire it. |

## Hardware & Safety Concept: Race Conditions and Critical Sections
A race condition occurs when multiple threads attempt to read and write to the same variable concurrently. For example, if both Core 0 and Core 1 read `balance = 100` at the same time, Core 0 calculates `100 + 20 = 120`, and Core 1 calculates `100 - 5 = 95`. Whichever core writes back last will overwrite the other's transaction, causing data loss. Mutexes define **critical sections** that restrict variable access to a single thread at a time, ensuring data integrity.

## Try This! (Challenges)
1. **Critical Limit Switch**: Sound a warning buzzer on GP14 if the balance falls below $50 (low balance alert).
2. **Double Deposit Protection**: Add a lock timeout that rejects deposit requests if the button is pressed twice within 500 ms.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Thread stops executing (Deadlock) | Lock acquired twice | Ensure that every call to `acquire()` is followed by a call to `release()`. Never call `acquire()` inside a block that already holds the lock. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [16 - Button ON/OFF](../../beginner/16-button-on-off.md)
- [181 - Pico Multithreaded Core Execution](181-pico-multithreaded-core-execution.md)
