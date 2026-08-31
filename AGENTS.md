# SiFli-SDK

This file provides guidance to agents when working with code in this repository.

## Overview

SiFli SDK is an embedded firmware development framework built on RT-Thread RTOS, targeting SiFli Technologies' SoC family. The SDK supports five chip families: SF32LB52X, SF32LB55X, SF32LB56X, SF32LB57X, and SF32LB58X, with some supporting multi-core configurations (HCPU, LCPU and ACPU).

## Build System

The build system uses **SCons** with **Kconfig** for configuration. All builds must run inside a properly initialized environment.

### Environment Setup (Windows PowerShell)

```powershell
./export.ps1
```

### Environment Setup (Linux/macOS)

```bash
source ./export.sh
```

### Building a Project

Navigate to an example's project directory which contains file `SConstruct` and `proj.conf`, then:

#### Build for a specific board
```
scons --board=<board_name> -j8
```
The artifacts are generated under directory `build_<board_name>`. 

#### Interactive configuration menu

```bash
sdk.py menuconfig --board=<board_name>
```

Example board names: `eh-lb551_hcpu`, `eh-lb563_hcpu`, `ec-lb583_hcpu`, `sf32lb52-lcd_n16r8_hcpu` for HCPU core.


### Configure a Board
Navigate to a board directory which contains file `board.conf` (`HCPU` and `LCPU` have separate folders), then type

```
sdk.py menuconfig
```

E.g., enter `customer/boards/sf32lb52-lcd_a128r16/hcpu` and execute `sdk.py menuconfig` to configure core `HCPU` of board `sf32lb52-lcd_a128r16`,
enter `customer/boards/sf32lb52-lcd_a128r16/lcpu` and execute `sdk.py menuconfig` to configure core `LCPU` of board `sf32lb52-lcd_a128r16`,


### Project Structure

Each example project contains:
- `SConscript` / `SConstruct` — SCons build definitions
- `Kconfig` / `Kconfig.proj` — configuration options
- `proj.conf` — default configuration values
- `rtconfig.py` — toolchain/compiler settings
- `<board_name>/ptab.json` or `<board_name>/ptab.yaml` — board-specific partition table

#### Standalone Application Projects

SiFli applications use a generic-project model that separates the application from the SDK and board implementation. `my_project/watch` is the local reference for this layout: the application directory contains only its own `project/` build/configuration files, `src/` code and resources, and documentation. It does not copy the SDK's shared `drivers/`, `middleware/`, `rtos/`, `external/`, or board implementations into the application.

- Run the SDK's `export.ps1` or `export.sh` first. The exported `SIFLI_SDK` path lets an application live anywhere, including outside the SDK repository.
- Build from the application's `project/` directory. Its `SConstruct` calls `PrepareEnv()`, while its `SConscript` imports the SDK root `SConscript` through `SIFLI_SDK` with `duplicate=0` and adds the application's `../src/SConscript` separately.
- Keep only application-owned configuration in `Kconfig.proj` and the minimal `proj.conf`. Add `<board_name>/` or `<chip_name>/` files only for application-specific overrides such as `proj.conf`, partition tables, memory maps, or linker scripts; continue to use the canonical board and SDK files for everything else.
- Treat `build_<board_name>/` as generated output. Its `.config`, `rtconfig.h`, linker/partition files, and mirrored `sifli_sdk/` object tree are produced for that board and are not application source files. Never edit or copy code back from this directory.
- When adding a new application project, copy or create only this minimal application skeleton. Do not vendor shared SDK modules into the new project. If shared behavior must change, edit the canonical SDK source; if behavior is application-specific, implement it under the application's `src/` or project override files.

For generated configuration and project overrides, priority is: application board override, application chip override, application common project file, SDK board configuration, then Kconfig defaults.

#### Dual-Core Projects

Some projects has user-defined LCPU firmware project, project structure lo like below

```
example/<name>/project/hcpu/   ← HCPU firmware project
example/<name>/project/lcpu/   ← LCPU firmware project
```

There's no need to build LCPU firmware separately. Just enter hcpu directory and run build command with HCPU board name, e.g. `sf32lb56-lcd_n16r12n1_hcpu` (suffix `_hcpu` could be omitted), it would build LCPU firmware automatically as LCPU is added as child project (by command `AddChildProj` in `SConstruct`).



### Configuration System

SDK uses Kconfig for configuration (Linux kernel style):
- Default values from Kconfig files
- Board-specific overrides in `board.conf`
- Project-specific overrides in `proj.conf`
- Final `.config` and `rtconfig.h` generated in `project/build_*/` during compilation

Configuration priority: `proj.conf` > `board.conf` > Kconfig defaults

For details read [build_and_configuration](docs\source\en\app_development\build_and_configuration.md)

## Repository Structure

```
customer/         Board Support Package (board configs, peripheral drivers)
  boards/         Per-board configuration and initialization files (board.h, pin assignments)
  peripherals/    Board-level peripheral drivers

drivers/
  cmsis/          Chip register headers, startup files, linker scripts
    sf32lb52x/    Per-SoC: startup_<chip>.c, <chip>.h register maps, *.sct/*.ld
    sf32lb55x/    (same pattern for each supported SoC)
    ...
  hal/            HAL implementation (OS-independent)
  Include/        Public HAL headers
  ll/             Low-level utilities

docs/             SDK Documentation

rtos/
  rtthread/       RT-Thread core + components (finsh shell, ulog, etc.)
    bsp/sifli/    RT-Thread device driver adapters (wraps HAL for OS use)
  os_adaptor/     OS abstraction layer (used by middleware)

middleware/       In-house components (bluetooth, audio, dfu, boot, crypto, etc.)
external/         Third-party libraries (LVGL v8/v9, mbedTLS, CherryUSB, FFmpeg, etc.)
example/          Example projects organized by feature category
tools/            Build tools, flash utilities, image tools, autotest scripts
tests/            Python unit tests for build tooling (flash table generation, etc.)
```

## Architecture Layers

```
Application / Example
        ↓
Middleware (bluetooth stack, audio, DFU, app_fwk, crypto...)
        ↓
RT-Thread RTOS
  ├── RT-Thread Device Drivers
  └── RT-Thread Components
        ↓
HAL (drivers/hal/ — no OS dependency)
        ↓
CMSIS / SoC registers (drivers/cmsis/<soc>/)
        ↓
Hardware (Cortex-M33 HCPU, optional low-power LCPU co-processor)
```

**HAL vs RT-Thread Device Drivers**: HAL is OS-independent and manages interrupts directly. RT-Thread device drivers sit on top of HAL, expose a standard `rt_device_t` interface, and manage ISR registration through RT-Thread. Applications should prefer device drivers unless writing OS-independent code.

## SiFli GUI API and Demo Precedence

- Before implementing or modifying GUI behavior, first search the existing SiFli applications, examples, and demos for the closest working implementation, and use it as the primary reference.
- Prefer SiFli-provided adapted or optimized GUI interfaces (including LVSF, resource/font managers, and application-framework helpers) over calling the underlying raw LVGL interfaces when an equivalent SiFli interface exists.
- Do not assume a raw LVGL interface behaves or renders identically on SiFli hardware. Check existing SiFli demos especially for fonts and text rendering, resources, input handling, animations, screen transitions, and object lifecycle management.
- Fall back to a raw LVGL interface only when no suitable SiFli interface or demo implementation exists, and keep that fallback localized.

## Coding Style
- `.clang-format` for C code

