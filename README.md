# Kitchen Safety Heat and Gas Monitoring System

An embedded C project for an **LPC21xx (ARM7)** microcontroller that watches kitchen temperature and gas levels, raises a buzzer and LED alarm when either becomes unsafe, and logs the most recent event with a timestamp. A keypad-driven, password-protected menu lets the user adjust the clock, the alarm limits and the password.

**Vector ID shown on splash screen:** `V25HE11M4`

---

## Features

- **Temperature monitoring** with an LM35 sensor (reads in °C, with a °F helper).
- **Gas monitoring** with an MQ2 sensor on an ADC channel.
- **Audible and visual alarm:** buzzer and LED switch on when a limit is crossed, and an `ALERT!!` message is shown on the LCD for about 2.5 s.
- **Last-event log:** the time and value of the most recent alarm is stored and re-displayed as a pop-up every 10 s for 3 s.
- **Real-time clock:** time and date are always shown on the 16x2 LCD.
- **Silence switch (Switch2):** mutes the buzzer and LED while pressed.
- **Settings menu (Switch1, hardware interrupt):** protected by a 4-digit password, with:
  1. Edit RTC (hour, minute, second, date, month, year)
  2. Edit set-points (temperature limit, gas limit)
  3. Change password
  4. Exit
- **Security:** masked password entry (`*`), 3 wrong attempts triggers a 10 s `System Locked` countdown.
- **Startup splash:** Vector ID screen, then a scrolling project title.

---

## Hardware

| Component | Details |
|---|---|
| MCU | LPC21xx family (ARM7), `lpc21xx.h` / `lpc214x.h` |
| Clocks | FOSC = 12 MHz, CCLK = 60 MHz (PLL x5), PCLK = 15 MHz |
| Display | 16x2 character LCD, 8-bit mode |
| Keypad | 4x4 matrix keypad |
| Temperature sensor | LM35 (10 mV per °C) |
| Gas sensor | MQ2 (analog output) |
| Outputs | Buzzer, LED |
| Inputs | Switch1 (interrupt), Switch2 (silence) |
| Clock source | RTC on the 32.768 kHz external crystal |

### Pin Map

| Function | Pin(s) | Notes |
|---|---|---|
| LCD data D0-D7 | P0.8 - P0.15 | |
| LCD RS / RW / EN | P0.16 / P0.17 / P0.18 | |
| LM35 output | P0.28 (AIN1, ADC channel 1) | |
| MQ2 output | P0.29 (AIN2, ADC channel 2) | |
| Buzzer | P0.27 | `BUZZER_PIN` |
| LED | P0.25 | `LED_PIN` |
| Switch1 | P0.1 (EINT0) | Interrupt, VIC channel 14 |
| Switch2 | P0.3 | Active-low |
| Keypad rows | P1.16 - P1.19 | Outputs |
| Keypad columns | P1.20 - P1.23 | Inputs |

### Keypad Layout

```
 7   8   9   /
 4   5   6   *
 1   2   3   -
 C   0   :   +
```

- Digits `0-9` are entered as numbers.
- `C` deletes the last digit.
- Any other key (for example `:`) confirms the entry.

---

## Default Settings

Defined at the top of `main.c`:

| Setting | Default | Meaning |
|---|---|---|
| `TEMP_LIMIT` | 40 | Alarm above 40 °C |
| `GAS_LIMIT` | 300 | Alarm above this raw ADC reading (0-1023) |
| `DEFAULT_PASSWORD` | 1234 | Can be changed from the menu |
| `MENU_WAIT_TIME` | 30000 ms | Menu times out after 30 s without a key |
| `POPUP_EVERY` | 10000 ms | Last-event pop-up interval |
| `POPUP_FOR` | 3000 ms | Pop-up duration |
| `SPLASH_HOLD_MS` | 2000 ms | Vector ID screen hold time |

The temperature limit, gas limit and password can all be changed at run time from the menu. They are held in RAM and are **not** saved across power cycles.

---

## LCD Screens

**Normal screen**
```
HH:MM:SS T:25°C
DD/MM/YYYY S:0
```
`S:` shows the gas status, 0 = safe and 1 = unsafe.

**Alert screen** (about 2.5 s, buzzer and LED on)
```
ALERT!!
TEMP IS HIGH!      (or GAS IS HIGH!)
```

**Last-event pop-up** (3 s, every 10 s once an event has occurred)
```
H:M:S T:45°C       (temperature event)
H:M:S S:1          (gas event)
```

**Menu**
```
1.RTC 2.SETPT<sec>
3.PASS 4.EXIT
```
The number at the end of line 1 is the remaining time before the menu closes.

---

## How It Works

1. **Startup:** initialises the LCD, ADC, gas pin, RTC, keypad, alarm pins, the millisecond timer and the Switch1 interrupt. The RTC is loaded with a start time and date, then the splash screens are shown.
2. **Main loop:**
   - If Switch1 raised the menu flag, run the password prompt and menu.
   - Read temperature (`LM35tC`) and gas (`read_gas_raw`).
   - `check_for_danger()` detects a new transition from safe to unsafe, logs the event, turns on the alarm and shows the alert.
   - While a limit is exceeded, the buzzer and LED stay on unless Switch2 is held.
   - Draw the normal screen, or the last-event pop-up when its timer fires.
3. **Edge-triggered logging:** `was_temp_high` and `was_gas_high` ensure each danger is logged once when it starts, not on every loop pass.
4. **Timekeeping:** Timer0 with prescaler 14999 gives a 1 ms tick (`time_now()`), used for the menu timeout and pop-up timing.

### Menu Flow

```
Switch1 press -> Enter Password -> (3 wrong -> 10 s lock, then retry)
                      |
                      v
        1. RTC      -> pick H/M/S/Date/Month/Year -> enter value -> Saved!
        2. SETPT    -> 1.TEMP (0-200) or 2.GAS (0-1023) -> Saved!
        3. PASS     -> current -> new -> confirm -> Saved!
        4. EXIT / 30 s timeout -> back to normal screen
```

---

## Project Structure

| File | Purpose |
|---|---|
| `main.c` | Application logic: alarms, splash, event log, menu, main loop |
| `types.h` | Short type names (`u8`, `u32`, `s32`, `f32`, ...) |
| `defines.h` | Bit-manipulation macros |
| `delay.c/.h` | Software delays (`delay_us`, `delay_ms`, `delay_s`) |
| `pin_connect_block.c/.h` | `cfgportpin()` helper for PINSEL configuration (port 0) |
| `ADC.c/.h`, `ADC_defines.h` | ADC init and read (10-bit, 3 MHz ADC clock) |
| `LM35.c/.h` | LM35 temperature in °C and °F |
| `LCD.c/.h`, `LCD_defines.h` | LCD driver: commands, strings, integers, floats, custom characters |
| `KPM.c/.h`, `KPM_defines.h` | 4x4 keypad scan and number input |
| `RTC.c/.h` | On-chip RTC set/get and LCD display helpers |

### Key Driver Functions

- **ADC:** `Init_ADC()`, `Read_ADC(channel, &digital, &voltage)` (voltage = 3.3 V x value / 1023)
- **LM35:** `LM35tC()` returns voltage x 100, `LM35tF()` converts to Fahrenheit
- **LCD:** `Init_LCD()`, `WRITE_LCD_CMD()`, `WRITE_LCD_DATA()`, `StrLCD()`, `U32LCD()`, `S32LCD()`, `F32LCD()`, `BuildcgRAM()`
- **Keypad:** `Init_KPM()`, `KeyScan()` (blocking), `ColScan()` (non-blocking check), `ReadNum()`
- **RTC:** `RTC_Init()`, `Set/GetRTCTimeInfo()`, `Set/GetRTCDateInfo()`, `Set/GetRTCDay()`

---

## Build and Run

1. Create a project in **Keil uVision** (or a compatible ARM7 toolchain) targeting your LPC21xx device.
2. Add all `.c` files and keep the headers in the same folder.
3. Build and flash the generated hex file with Flash Magic or your programmer.
4. Connect the hardware as per the pin map and power up. The splash screen appears, followed by the live monitoring screen.

---

## Notes and Known Issues

- **RTC is reset on every boot:** `main()` calls `SetRTCTimeInfo(9, 37, 0)` and `SetRTCDateInfo(24, 9, 2026)` at startup, overwriting the running clock. Remove these lines (or only set on first start) if you want the time to survive power cycles with a battery backup.
- **Comment mismatches in `main.c`:** the comments beside `BUZZER_PIN` and `LED_PIN` mention P0.22 and P0.30, but the values in code are P0.27 and P0.25. The code values are the ones that apply.
- **Comments mention `=`:** the keypad has no `=` key. The confirm key is `:` (or any non-digit, non-`C` key).
- **Software delays:** `delay.c` uses busy-wait loops whose timing depends on compiler optimisation, so values are approximate.
- **`CLRBIT` macro in `defines.h`** has an operator-precedence problem (`~1<<BITPOS`). It is unused in this project, but should be written `~(1<<BITPOS)` if used.
- **Duplicate defines:** `FOSC`, `CCLK` and `PCLK` appear in both `RTC.h` and `ADC_defines.h`, which may produce redefinition warnings.
- **Unused code:** `LM35NT()` is declared in `LM35.h` but commented out in `LM35.c`; the `DisplayRTC*` functions in `RTC.c` are not used by `main.c`.
- The gas status on the normal screen is a simple 0/1 flag; the raw MQ2 value display is commented out in `show_normal_screen()`.

---

## Author

Project by Vector ID **V25HE11M4**.
