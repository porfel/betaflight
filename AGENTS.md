# AGENTS.md

## Cursor Cloud specific instructions

### Overview

Betaflight is an embedded C firmware for drone flight controllers. The build system is GNU Make. There is no web UI, no running services, and no package manager lockfile — the project compiles to bare-metal `.hex`/`.bin`/`.uf2` firmware images.

### Key commands

| Task | Command |
|---|---|
| Build firmware for a target | `make TARGET=<name>` (e.g. `make TARGET=STM32F4DISCOVERY`) |
| Build SITL desktop simulator | `make SITL` |
| Run unit tests | `make test` |
| Run all tests (incl. per-target) | `make test-all` |
| Run a single test | `make test_<name>` (e.g. `make test_pid_unittest`) |
| Static analysis checks | `make checks` |
| Cppcheck | `make cppcheck` |
| Clean build artifacts | `make clean` |
| Clean test artifacts | `make test_clean` |
| Install ARM SDK | `make arm_sdk_install` |
| Hydrate target configs | `make configs` |
| Print version | `make version` |

### Non-obvious gotchas

- **ARM GCC toolchain**: The ARM cross-compiler (v13.3.1) is installed into `tools/` via `make arm_sdk_install`. The build system detects it automatically when present. Do NOT use a system `arm-none-eabi-gcc` unless it matches the exact version.
- **Clang-18 for tests**: Unit tests require `clang-18` / `clang++-18` with `libc++-18-dev`, `libc++abi-18-dev`, `libclang-rt-18-dev`, `libstdc++-14-dev`, and `libblocksruntime-dev`. The test Makefile hardcodes `clang-18`.
- **Git submodules**: `src/config` (target board configs) and `lib/main/pico-sdk` (Pico SDK) are git submodules. They must be initialized (`git submodule update --init --recursive`) before building.
- **SITL binary**: After `make SITL`, the binary is at `obj/betaflight_<version>_SITL`. It creates an `eeprom.bin` in the working directory at runtime — clean it up if needed.
- **SITL UDP ports**: The SITL simulator uses UDP ports 9001-9004 for communication with external physics simulators (Gazebo, RealFlight).
- **No lint command**: There is no dedicated `make lint`. Use `make checks` for build sanity checks and `make cppcheck` for static analysis.
