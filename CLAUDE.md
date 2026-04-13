# CLAUDE.md

Guidance for Claude Code (and other AI coding assistants) working in this
repository. This file complements `AGENTS.md` with more depth on codebase
layout, build system, and conventions.

## Project overview

Betaflight is open-source flight controller firmware (GPLv3) for multirotors
and fixed-wing craft. It is an embedded C codebase (C17, `-std=gnu17`) that
targets ARM Cortex-M MCUs — primarily STM32 F4/F7/G4/H5/H7/N6, with additional
support for APM32, AT32, PICO (RP2350), and a SITL/SIMULATOR build.

- Primary language: C (C17), with C++ used only for GoogleTest unit tests
- Build system: GNU `make` invoked from the repository root
- Toolchain: `arm-none-eabi-gcc` installed via `make arm_sdk_install` into
  `tools/` (cached by CI). The `mk/tools.mk` hash-keys the cache.
- LTO is enabled (`-flto=auto -fuse-linker-plugin -ffast-math`) with
  per-file optimisation policy (default `-O2`, speed-optimised `-Ofast`,
  size-optimised `-Os`).
- Warnings are errors: `-Wall -Wextra -Werror -Wunsafe-loop-optimizations
  -Wdouble-promotion -Wold-style-definition`. CI adds `EXTRA_FLAGS=-Werror`.

## Repository layout

```
betaflight/
├── Makefile                Root make entrypoint (targets, firmware build)
├── AGENTS.md               Short agent-oriented notes (read alongside this file)
├── README.md               Project overview, release schedule
├── CONTRIBUTING.md         Contribution pointers (see betaflight.com/docs/development)
├── .devcontainer/          Dockerfile + VS Code devcontainer config
├── .github/workflows/      ci.yml (build matrix), pr.yml, push.yml, build-release.yml
├── mk/                     Makefile fragments (tools, os, verbosity, preprocess, checks, source)
├── lib/main/               Vendored third-party code (CMSIS, STM32 HAL/LL, USB stacks,
│                           MAVLink, FatFS, googletest, dyad, pico-sdk, etc.)
│                           DO NOT modify unless the task explicitly requires it.
├── images/                 Logos / assets
└── src/
    ├── main/               Platform-independent firmware sources (most edits land here)
    ├── platform/           Per-MCU-family implementations (STM32, APM32, AT32, PICO, SIMULATOR, common)
    ├── target/             (inside src/main) common pre/post headers shared by target.h files
    ├── test/               GoogleTest-based host unit tests (Makefile-driven)
    ├── utils/              Helper scripts (e.g. dfuse-pack.py)
    └── config/             Git submodule: betaflight/config (board/target configs)
```

### `src/main` — platform-independent firmware

- `main.c`              Entry point (`main()` → init → scheduler loop)
- `fc/`                 Flight controller core (tasks, scheduler hooks, runtime
                        config, arming, RC input handling, core loop)
- `flight/`              Flight algorithms: `pid.c`, `imu.c`, `mixer.c`,
                         `failsafe.c`, `position.c`, `rpm_filter.c`,
                         `dyn_notch_filter.c`, `servos.c`. Multirotor and
                         fixed-wing (`_wing`) variants split several modules:
                         `alt_hold_*`, `autopilot_*`, `gps_rescue_*`,
                         `pos_hold_*`.
- `sensors/`            Gyro, accelerometer, barometer, compass, battery,
                        current, voltage, ESC sensor, optical flow, rangefinder
- `drivers/`            Hardware-abstracted drivers (bus_spi/i2c/octospi/quadspi,
                        dma, timer, dshot, ws2811, accgyro/, barometer/,
                        compass/, flash/, rx_spi/, etc.). Platform-specific
                        implementations live under `src/platform/*`.
- `rx/`                 Receiver protocols (CRSF, SBUS, GHST, SRXL2, ELRS,
                        FPort, Spektrum, iBus, Jeti, PWM/PPM, SPI-based RX)
- `telemetry/`          Telemetry protocols (CRSF, FrSky, HoTT, iBus, LTM,
                        MAVLink, SmartPort)
- `io/`                 I/O subsystems (serial, USB CDC/MSC, beeper, LED strip,
                        flashfs, GPS, VTX: SmartAudio/Tramp/MSP/RTC6705, OSD
                        displayports, RC Device/camera control)
- `osd/`                On-screen display rendering and element layout
- `cms/`                On-screen configuration menu system
- `msp/`                MultiWii Serial Protocol server (`msp.c`, `msp_box.c`,
                        `msp_serial.c`, `msp_protocol*.h`)
- `cli/`                Command-line interface (`cli.c`, `settings.c`)
- `blackbox/`           Flight log recorder (to flash or SD card)
- `scheduler/`          Cooperative task scheduler (`scheduler.c`)
- `pg/`                 "Parameter Group" (`PG_*`) — persisted config storage
                        with defaults and versioning
- `config/`             Config streamer, EEPROM-style storage, feature bits,
                        simplified tuning
- `common/`             Math/utility (filters, CRC, maths/vector, printf,
                        streambuf, SDFT, string helpers, time)
- `build/`              Build metadata (version header, atomic helpers, debug)
- `target/`              `common_pre.h` / `common_post.h` /
                         `common_defaults_post.h` included by per-target
                         `target.h` files (see `src/platform/*/target/*/target.h`)

### `src/platform/*` — MCU-specific code

Each platform directory holds an `mk/<family>.mk` include, MCU HAL glue,
peripheral drivers (ADC/DMA/I2C/SPI/timer/USB/etc. per family), and a
`target/<TARGET_NAME>/` subtree with `target.h` and `target.mk` per board.
Cross-platform driver headers in `src/main/drivers/*_impl.h` define the
interface implemented by the platform-specific `.c` files. Ongoing
refactoring work (see recent commits) moves `#ifdef STM32xxx` platform
splits out of `src/main` into `src/platform/*` headers behind feature flags
like `ENABLE_AFATFS_DMA_CACHE`, `ENABLE_OVERCLOCK_xxx_MHZ`. Prefer that
pattern when touching platform-conditional code.

### `src/test/unit` — host unit tests

- GoogleTest-based `.cc` files, compiled natively on the build host.
- Per-test source/define lists in `src/test/Makefile`
  (e.g. `pid_unittest_SRC`, `pid_unittest_DEFINES`).
- `*_EXPAND` tests run once per target (representative run: once per test).

## Build system

### Build a firmware target

Run from the repo root. The configs submodule must be hydrated:

```
git submodule update --init --recursive   # one-time, pulls src/config and pico-sdk
make arm_sdk_install                      # one-time, installs toolchain under tools/
make configs                              # hydrate config definitions
make TARGET=<TARGET_NAME>                 # default (fwo: .hex/.uf2/.exe by platform)
make hex TARGET=<TARGET_NAME>             # explicit hex
make uf2 TARGET=<TARGET_NAME>             # for PICO targets
make TARGET=<TARGET_NAME> clean           # clean a single target
make clean                                # full clean of obj/
```

`make targets` prints every `BASE_TARGETS` value (board names) and the CI
subset. `make help` prints every documented `## ...` rule in the Makefile.
Target discovery scans `src/platform/*/target/*/target.mk`.

### Useful flags

- `DEBUG=INFO|GDB`               Build with debug symbols (`GDB` lowers opt)
- `EXTRA_FLAGS=-Werror`          Matches CI
- `OPTIONS="FOO BAR"`            Adds `-DFOO -DBAR` to compilation
- `V=1`                          Verbose command echo
- `EXST=yes`                     External-storage bootloader build
- `RAM_BASED=yes`                Execute from RAM
- `REV=yes`                      Append git revision to output filename
  (alias: `make <TARGET>_rev`)

### Tests and static analysis

```
make test                 # full GoogleTest suite (host)
make test-representative  # run each test once (per-target expansions collapsed)
make test-all             # include all per-target expansions
make test_<name>          # single test, e.g. make test_pid_unittest
make test_help            # list available tests
make cppcheck             # static analysis across src/main
```

### Flashing helpers (local dev, not CI)

`make <TARGET>_flash`, `make tty_flash`, `make dfu_flash`, `make st-flash`,
`make unbrick`, `make openocd-gdb`. Use the devcontainer (`.devcontainer/`)
for a reproducible toolchain.

## Coding conventions

- Follow the coding style at https://www.betaflight.com/docs/development/CodingStyle
  (4-space indent, K&R braces, `snake_case` filenames, `camelCase` for
  functions/variables, `UPPER_CASE` for macros/enum members, `lowerCamelCase`
  typedefs suffixed with `_t`).
- Headers use `#pragma once`.
- Files start with the standard GPLv3 Cleanflight/Betaflight notice (see any
  existing source file for the boilerplate).
- Parameter groups: new configurable settings go in `src/main/pg/` with a
  `PG_REGISTER_*` macro and a CLI binding in `src/main/cli/settings.c`.
- Feature flags: prefer `USE_xxx` compile-time defines and
  `ENABLE_xxx_yyy` platform capability flags over `#ifdef STM32H7` style
  MCU checks. When adding platform conditionals, put them in
  `src/platform/<family>/include/platform.h` (or the appropriate platform
  header), not in `src/main`.
- Multirotor vs. fixed-wing: many flight modules have `_multirotor.c` and
  `_wing.c` variants. Shared declarations live in the non-suffixed header.
- Avoid floating-point in ISRs and hot paths where not already present; many
  files are compiled speed-optimised and some are size-optimised — keep that
  in mind and do not broaden the optimisation set lightly.
- Do not edit vendored code in `lib/main` unless the task requires it.
- Keep warnings clean — `-Werror` is on.

## Git, branches, and CI

- Work happens on feature branches; PRs target `master`. Release branches are
  named `*-maintenance`.
- GitHub Actions runs `ci.yml` on push/PR: full target matrix build with
  `EXTRA_FLAGS=-Werror` plus the unit test suite. Don't ignore new
  warnings; they will fail CI.
- Builds are expensive — for local validation prefer one relevant target
  plus `make test-representative` (or `make test_<name>` for a focused
  change) instead of the whole matrix.
- Submodules: `src/config` (configs) and `lib/main/pico-sdk` (Pico SDK).
  Always hydrate with `--recursive` or `make configs` before building.

## When making changes (high-signal validation)

1. Build at least one representative firmware target that exercises the
   changed code path:
   - `make TARGET=<some_stm32_target>` for generic core/firmware changes
   - A PICO or N6 target if you touched platform code for those MCUs
2. Run relevant unit tests: `make test_<name>` for the subsystem, or
   `make test-representative` for broader confidence.
3. For docs-only or comment-only edits, a lightweight sanity check is fine.
4. For UI/OSD/CMS behaviour that has no automated coverage, state in the PR
   description exactly which build + test commands were run.

## Assistant-specific guidance

- Use this repo's dedicated search tools (Glob/Grep) rather than shelling
  out to `find`/`grep`. Ripgrep-backed Grep is fast across the whole tree.
- Before proposing a code change, read the file — do not guess existing
  signatures, especially across `src/main` ↔ `src/platform` boundaries.
- Keep diffs minimal and focused on the task. Do not opportunistically
  refactor, reformat, or rename.
- Don't push to `master`. Develop on the designated feature branch; push
  with `git push -u origin <branch>`.
- Don't open a PR unless the user explicitly asks for one.
- Respect the GitHub MCP scope restriction (only `porfel/betaflight`).
