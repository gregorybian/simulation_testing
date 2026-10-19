# sunsim — Simulating STR in Renode, from scratch

The method is a loop: **run → see where it gets stuck → read why → write the smallest fix → run
again.** Each stage below is one turn of that loop.

STR's clock setup is simpler than TEL's (no RTC clock config), so you need **fewer stubs** than TEL
did. You'll also find out what happens with the IWDG and hex display when they're left unmodeled.
That's part of the experiment.

> Answer key: `~/Desktop/UBC_Solar/renode_sims/` is the working TEL version. Only look if stuck.

---

## Stage 0: Folder layout

Use one shared set of peripheral models that every board reuses, plus one `.resc` per board:

```bash
mkdir -p ~/Desktop/UBC_Solar/sunsim/{peripherals,boards,tests}
cd ~/Desktop/UBC_Solar/sunsim
```

```
sunsim/
  GUIDE.md       this file
  peripherals/   Python stubs (rcc, plain registers, ...) shared by all boards
  boards/        str.resc (later drd.resc, tel.resc, ...)
  tests/         robot files, later
```

**Always run Renode from inside `sunsim/`.** The paths in your `.resc` will be relative
(`@peripherals/...`), and those only resolve against the folder you start from.

---

## Stage 1: Bare `.resc` and see it fail

Create `boards/str.resc` with only the essentials:

```
# Steering wheel (STR) on a virtual STM32F103
using sysbus
mach create "str"
machine LoadPlatformDescription @platforms/cpus/stm32f103.repl

sysbus LoadELF @/home/gregorybian/Desktop/UBC_Solar/firmware_v4/firmware/components/str/build/Debug/str.elf
showAnalyzer usart5
```

What each line does:

| Line | Meaning |
|---|---|
| `mach create` | Makes an empty virtual board. |
| `LoadPlatformDescription` | Puts an STM32F103 on it, using Renode's built-in model (`renode/platforms/cpus/stm32f103.repl`). |
| `LoadELF` | Flashes your firmware. |
| `showAnalyzer` | Opens a window showing UART5 output. |

Run it:

```bash
renode boards/str.resc
```

Then, in the Monitor:

```
start
pause
sysbus FindSymbolAt `cpu PC`
sysbus FindSymbolAt `cpu LR`
```

- `cpu PC` is where the CPU is now.
- `cpu LR` is roughly who called it.
- `FindSymbolAt` turns an address into a function name using your ELF.

> **`FindSymbolAt` prints nothing?** You probably skipped `start`. Before the machine starts, PC is
> `0x0` (it's only loaded from the vector table at start), and an address with no symbol prints an
> empty line. Run `cpu PC` on its own first to see the raw address.

**Expected:** PC is in `Error_Handler` and LR is in `SystemClock_Config`. Also scroll the log for
warnings like `unimplemented register RCC:...`. Those warnings are the clue.

**To rerun after editing the `.resc`:** type `Clear` then `include @boards/str.resc`, or just
restart Renode.

---

## Stage 2: Your first Python peripheral (plain storage)

**Why it fails:** Renode's F103 model has **no RCC**, the clock controller. Reads of RCC registers
come back as fixed defaults. The repl fakes one register (`RCC_CR`), which you can see at the bottom
of `stm32f103.repl`.

Start with the simplest possible peripheral: memory that remembers what was written. Create
`peripherals/stub_regs.py`:

```python
# Plain register storage: writes stick, reads return the last write (0 if never written).
if request.IsInit:
    regs = {}                     # offset -> value, persists between calls
elif request.IsWrite:
    regs[request.Offset] = request.Value
elif request.IsRead:
    request.Value = regs.get(request.Offset, 0)
```

How this file works:

- Renode runs the whole file **every time** the CPU touches the peripheral's address range.
- `request` describes that one access:
  - `IsInit`: true once at startup.
  - `IsRead` / `IsWrite`: what kind of access this is.
  - `Offset`: bytes from the base address.
  - `Value`: the value written, or the value you return on a read.
- Variables you create at init, like `regs`, survive between calls.

Attach it over the RCC. RCC is at `0x40021000` and is `0x400` bytes long, per the RM0008 memory
map. Put this **before** `LoadELF`:

```
machine PyDevFromFile @peripherals/stub_regs.py 0x40021000 0x400 True "rcc"
```

Run again. **It still fails.** The reason is the key lesson:

HAL writes `HSEON = 1` and then **polls `HSERDY` until it becomes 1**. On real silicon the hardware
sets `HSERDY` once the crystal has started. Plain storage only remembers what software wrote, so
`HSERDY` stays 0 forever, the HAL timeout expires, and you land in `Error_Handler`.

---

## Stage 3: A smarter RCC (ready bits follow enable bits)

Copy `stub_regs.py` to `peripherals/stm32f1_rcc.py`. Change the write branch so that when an "ON"
bit is written, its "RDY" bit is set too. These are the bits HAL waits on, from RM0008 §8.3:

| Register (offset) | Software sets | Hardware should set | Reset value |
|---|---|---|---|
| `RCC_CR` (0x00) | HSION bit 0, HSEON bit 16, PLLON bit 24 | HSIRDY bit 1, HSERDY bit 17, PLLRDY bit 25 | `0x00000083` |
| `RCC_CFGR` (0x04) | SW bits [1:0] (clock source select) | SWS bits [3:2] = copy of SW | `0` |
| `RCC_CSR` (0x24) | LSION bit 0 | LSIRDY bit 1 | `0x0C000000` |

Skeleton to fill in:

```python
CR, CFGR, CSR = 0x00, 0x04, 0x24

def mirror(offset, v):
    if offset == CR:
        # TODO: clear bits 1, 17, 25, then set each to match bits 0, 16, 24
        pass
    elif offset == CFGR:
        # TODO: bits [3:2] = bits [1:0]
        pass
    elif offset == CSR:
        # TODO: bit 1 = bit 0
        pass
    return v

if request.IsInit:
    regs = {CR: 0x83, CSR: 0x0C000000}      # reset values
elif request.IsWrite:
    regs[request.Offset] = mirror(request.Offset, request.Value)
elif request.IsRead:
    request.Value = regs.get(request.Offset, 0)
```

Hint for one bit: `v = (v & ~(1 << 17)) | (((v >> 16) & 1) << 17)` copies bit 16 into bit 17.

Point the RCC line at the new file and run again.

---

## Stage 4: Bit-banding

You'll probably still fail, with a log warning like
`WriteDoubleWord to non existing peripheral at 0x42420480`.

**What that is:** HAL turns on the LSI with a **bit-band** write. Cortex-M3 maps every bit of every
peripheral register to its own 32-bit address in `0x42000000+`, so writing 1 there sets exactly one
bit:

```
alias = 0x42000000 + (register_addr − 0x40000000) × 32 + bit × 4
```

**Exercise:** check that `0x42420480` is `RCC_CSR` (0x40021024) bit 0, i.e. LSION.

Renode's F103 model doesn't include bit-banding, so that write disappears. Renode already has a model
for it, so the fix is one line before `LoadELF`:

```
machine LoadPlatformDescriptionFromString "bitband: Miscellaneous.BitBanding @ sysbus <0x42000000, +0x2000000> { peripheralBase: 0x40000000 }"
```

---

## Stage 5: Flash wait states

Next, `HAL_RCC_ClockConfig` writes `LATENCY = 2` to `FLASH_ACR` (`0x40022000`) and **reads it back
to check**. Renode's model returns a fixed value, so the check fails.

Your `stub_regs.py` from Stage 2 is exactly the right fix, just at a new address. Write that line
yourself.

---

## Stage 6: CAN

Once the clocks pass, LR will point at `MX_CAN_Init`, because the F103 model has no CAN controller.
You don't need Python here. Renode's STM32F4 model has a bxCAN with identical registers, and the
F103's CAN interrupts are also IRQs 19–22:

```
machine LoadPlatformDescriptionFromString "can1: CAN.STMCAN @ sysbus <0x40006400, +0x400> { [0-3] -> nvic@[19-22] }"
sysbus WriteDoubleWord 0x40006400 0x00010000
```

The second line is a workaround. HAL asks the CAN controller to enter init mode while it's asleep.
Real chips accept that, but Renode's model ignores it, so this line starts the controller awake.

---

## Stage 7: Reaching the tasks, and what happens to the IWDG and hex display

Neither is modeled, and that's deliberate for now:

- **IWDG:** `HAL_IWDG_Init` only waits for its status register to read 0, which unmodeled registers
  already do. Refresh writes are ignored, so the watchdog never resets the sim. Expected to pass
  silently.
- **Hex (AS1115 on I2C1):** `StartHexDisplayTask` will try to talk to an I2C device that doesn't
  exist. This is the open question. Renode's I2C either:
  - reports "no ACK" quickly, so the task errors and carries on; or
  - never responds. In that case `HAL_MAX_DELAY` means that task spins forever, and because it runs
    at Normal priority it may **starve the Low-priority tasks** (steering outputs, diagnostics).

To check whether the scheduler is alive, run `start`, wait about 5 seconds, `pause`, then:

```
sysbus ReadDoubleWord `sysbus GetSymbolAddress "uwTick"`
```

That's the HAL millisecond counter, so after about 5 s of simulated time it should be about
`0x1388` (5000).

Then repeat `start` / `pause` / ``sysbus FindSymbolAt `cpu PC` `` a few times. If PC is always
inside I2C functions, the hex task is hogging the CPU. That's your cue to model the AS1115 next, or
temporarily skip that task.

---

## Cheat sheet

| Goal | Monitor command |
|---|---|
| Run / pause | `start` / `pause` |
| Where am I? | ``sysbus FindSymbolAt `cpu PC` `` |
| Who called? | ``sysbus FindSymbolAt `cpu LR` `` |
| Reload after edits | `Clear` then `include @boards/str.resc` |
| Log one peripheral's accesses | `sysbus LogPeripheralAccess rcc true` |
| Log every peripheral access | `sysbus LogAllPeripheralsAccess true` |
| More log detail | `logLevel -1` |
| List peripherals | `peripherals` |
| Read a variable | ``sysbus ReadDoubleWord `sysbus GetSymbolAddress "uwTick"` `` |
| Drive an input pin | `sysbus.gpioPortA OnGPIO 9 false` (active-low button pressed) |

## Next steps (after Stage 7)

1. Model the AS1115 hex display on I2C1 (address `0x00`).
2. Drive buttons from the Monitor, and add an LED model on `DEBUG_LED` (PC3).
3. Debug in VS Code: add `machine StartGdbServer 3333` to `str.resc` and a Cortex-Debug launch
   entry with `"servertype": "external"` and `"gdbTarget": "localhost:3333"`.
4. Automate with robot tests in `tests/`, run with `renode-test`.

## References

- [Renode docs: Monitor and script syntax](https://renode.readthedocs.io/en/latest/basic/monitor-syntax.html)
- [Renode docs: Describing platforms](https://renode.readthedocs.io/en/latest/basic/describing_platforms.html)
- [Renode docs: Using Python (Python peripherals)](https://renode.readthedocs.io/en/latest/basic/using-python.html)
- [Renode docs: Debugging with GDB](https://renode.readthedocs.io/en/latest/debugging/gdb.html)
- [Renode docs: Debugging with VS Code](https://renode.readthedocs.io/en/latest/debugging/vscode.html)
- RM0008 (STM32F103 reference manual): §3.3 memory map, §8.3 RCC registers
