<div align="center">
    <a href="https://github.com/Ultrawipf/OpenFFBoard">
        <img width="200" height="200" src="doc/img/ffboard_logo.svg">
    </a>
	<br>
	<br>
	<div style="display: flex;">
		<a href="https://discord.gg/gHtnEcP">
            <img src="https://img.shields.io/discord/704355326291607614">
		</a>
		<a href="https://github.com/Ultrawipf/OpenFFBoard/stargazers">
            <img src="https://img.shields.io/github/stars/Ultrawipf/OpenFFBoard">
		</a>
		<a href="https://github.com/Ultrawipf/OpenFFBoard/actions/workflows/build-firmware.yml">
            <img src="https://github.com/Ultrawipf/OpenFFBoard/actions/workflows/build-firmware.yml/badge.svg?branch=master">
		</a>
	</div>
</div>



# Open FFBoard
The Open FFBoard is an open source force feedback interface with the goal of creating a platform for highly compatible FFB simulation devices like steering wheels and joysticks.

This firmware is optimized for the Open FFBoard and mainly designed for use with DD steering wheels.
Remember this software is experimental and is intended for advanced users. Features may contain errors and can change at any time.

More documentation about this project is on the [hackaday.io page](https://hackaday.io/project/163904-open-ffboard).

The hardware designs are found under [OpenFFBoard-hardware](https://github.com/Ultrawipf/OpenFFBoard-hardware).

The GUI for configuration is found at [OpenFFBoard-configurator](https://github.com/Ultrawipf/OpenFFBoard-configurator).

These git submodules can be pulled with `git submodule init` and `git submodule update`

Updates often require matching firmware and GUI versions!

## Documentation
Documentation will be updated in the [GitHub Wiki](https://github.com/Ultrawipf/OpenFFBoard/wiki).

Available commands are listed on the [Commands wiki page](https://github.com/Ultrawipf/OpenFFBoard/wiki/Commands)

Code summary and documentation of the latest stable version is available as a [Doxygen site](https://ultrawipf.github.io/OpenFFBoard/doxygen/).

For discussion and progress updates we have a [Discord server](https://discord.com/invite/gHtnEcP).

### Extensions
The modular structure means you are free to implement your own main classes.
Take a look into the FFBoardMain and ExampleMain class files in the UserExtensions folder.
Helper functions for parsing CDC commands and accessing the flash are included.

The firmware is class based in a way that for example the whole main class can be changed at runtime and with it for example even the usb device and complete behavior of the firmware.

For FFB the motor drivers, button sources or encoders also have their own interfaces.

A unified command system supporting different interfaces is available and recommended for setting parameters at runtime. (see `CommandHandler.h` and the example mainclass)


### Copyright notice:
Some parts of this software may contain third party libraries and source code licenced under different terms.
The license applying to these files is found in the header of the file.
For all other parts in the `Firmware/FFBoard` folder the LICENSE file applies.

### Local fork changes — **2026-05-20**

The following items are **not** part of upstream OpenFFBoard; they are documented here so you can reproduce or revert them.

#### Custom pin remap (F407VG_DISCO)

The stock DISCO firmware maps the first three digital inputs (`DIN0`–`DIN2`, gamepad buttons D0–D2) to **PC15, PC14, and PC13**. On many boards those pins are tied to the **32 kHz crystal (LSE)** and **RTC** functions, so using them for momentary switches is awkward or impossible.

This fork moves only those three lines to general-purpose pins that are already broken out on the connector as **GP2, GP3, and GP1**. The higher digital inputs **DIN3–DIN7** are unchanged. `LocalButtons` is untouched: it still reads logical `DIN0`…`DIN7` in order; only the STM32 pin macros and `MX_GPIO_Init()` in the DISCO target were updated.

| Logical input | Old STM32 pin | New STM32 pin | Connector label (typical) |
| --- | --- | --- | --- |
| DIN0 / D0 | PC15 | **PB4** | GP2 |
| DIN1 / D1 | PC14 | **PB5** | GP3 |
| DIN2 / D2 | PC13 | **PD6** | GP1 |
| DIN3–D7 / D3–D7 | PE6 … PE2 | *(unchanged)* | — |

**Wiring:** digital inputs use internal **pull-up**; connect each button between the DIN (or GP) line and **GND** (normally-open, active low). Do not tie these lines to 5 V.

**Build:** `make MCU_TARGET=F407VG_DISCO` after editing `Firmware/Targets/F407VG_DISCO/Core/Inc/main.h` and `…/Core/Src/main.c`. Re-flash the generated `.hex`. If your hardware matches the official OpenFFBoard 1.2+ pinout instead, use `MCU_TARGET=F407VG`, not DISCO.

#### Axis / encoder position reset (“center here”)

Upstream already labels **PB2** as `BUTTON_A` with EXTI on the rising edge. On some harnesses that line is the same pad as **BOOT1** on the MCU.

This fork additionally polls **PB2** inside `FFBHIDMain::updateControl()` (`FFBHIDMain.cpp`): when the pin goes **inactive → active** (rising edge in software), it sets `control.resetEncoder`, which clears the axis encoder position to zero via `AxesManager::resetPosZero()` (same as “re-center wheel” in software — it does **not** cut motor torque).

| Item | Detail |
| --- | --- |
| **Pin** | **PB2** (`BUTTON_A`), DISCO / F407VG `main.h` |
| **Electrical** | **PB2** must read **low** when released and **high** when pressed. On this DISCO tree `BUTTON_A` is initialized with **`GPIO_NOPULL`** — use a **momentary NO** switch between **3.3 V** and **PB2**, and a **pull-down resistor** (e.g. 10k) from **PB2** to **GND** so the idle level is defined. (Other targets may use internal pull-down in CubeMX.) |
| **Behaviour** | Each **press** (0 → 1) recenters the logical axis position to 0. |
| **Not** | This is **not** the emergency stop. E-stop is **PD5** (`E_STOP`), active **low** (switch to GND). |
| **Boot caution** | If `BTNFAILSAFE` is defined for your target, **holding PB2 high at power-on** can select the failsafe main class. Only use this as a center button while the board is already running. |

**Code:** `Firmware/FFBoard/UserExtensions/Src/FFBHIDMain.cpp` (block guarded by `#ifdef BUTTON_A_Pin`).

