# AI Agent Guide for ArduPilot

ArduPilot is safety-critical autopilot software controlling real vehicles. Every change must be correct, tested, and reviewable.

---

## Build Commands

```sh
./waf configure --board sitl       # SITL (software-in-the-loop, for dev)
./waf copter                       # Build a vehicle: copter, plane, rover, sub, heli, antennatracker
./waf plane                        # ...
./waf --targets tests/test_math    # Build a specific unit test
./waf list_boards                  # List available boards
./waf clean                        # Clean current board
./waf distclean                    # Clean everything
```

Never run `waf` with `sudo`. Always call `./waf` from repo root.

---

## Testing

### SITL Autotest (Integration Tests)

The primary test system. Spawns a simulated vehicle with scripted flight scenarios.

```sh
# Run all tests for a vehicle (with rebuild)
Tools/autotest/autotest.py build.Copter test.Copter

# Run a specific test (with rebuild)
Tools/autotest/autotest.py build.Copter test.Copter.RTLYaw

# Plane tests
Tools/autotest/autotest.py build.Plane test.Plane

# Rover tests
Tools/autotest/autotest.py build.Rover test.Rover
```

Vehicle test suites: `Tools/autotest/arducopter.py`, `arduplane.py`, `rover.py`, `ardusub.py`, `antennatracker.py`, `blimp.py`, `helicopter.py`, `quadplane.py`, `sailboat.py`, `balancebot.py`.

### C++ Unit Tests (GTest)

Located in `libraries/<lib>/tests/`. Use `#include <AP_gtest.h>`.

### Pre-commit Hooks

```sh
pre-commit run --all-files         # Run all hooks against all files
pre-commit run                     # Run against staged files only
```

Hooks enforce: LF line endings, no large files, XML/YAML validity, codespell, astyle, flake8, ruff, serial-protocol gates. `modules/` and `build/` are excluded.

### Python Tests

`tests/` contains Python tests run by pytest in CI. Avoid code at global scope in `test_*.py` files — it executes during pytest's discovery process. Classes, functions, and `if __name__ == "__main__":` blocks are safe.

---

## Code Style (C++)

Enforced by astyle (`Tools/CodeStyle/astylerc`):

- 4 spaces, no tabs. Linux/K&R brace style (opening brace on same line).
- `#pragma once` header guards (not `#ifndef`).
- Always add braces on single-line blocks.
- **Format only the changed parts of files** — don't reformat entire files (breaks git blame).

`.editorconfig` mirrors these settings: 4-space indent, LF line endings, UTF-8, final newline.

Key conventions:

- Classes: `AP_` or `AC_` prefix, PascalCase. Methods: `snake_case`.
- Member variables: `_singleton`, `_primary`. Constants: `UPPER_SNAKE_CASE`.
- Compile-time flags: `AP_<NAME>_ENABLED`.
- Use `extern const AP_HAL::HAL& hal;` in `.cpp` files needing hardware access.
- `GCS_SEND_TEXT()` for user-facing messages (not `printf`).
- `AP_HAL::millis()` / `AP_HAL::micros()` (not platform-specific time).
- `is_zero()`, `is_positive()`, `is_negative()` over direct float comparisons.
- Wrap optional features in `#if AP_<FEATURE>_ENABLED` guards.
- Core components must never depend on compile-time optional components.

## Code Style (Python)

- New Python files must include `AP_FLAKE8_CLEAN` marker to opt into linting.
- flake8: max line length 127 (`.flake8`).
- `black` (line-length=120) only for `libraries/AP_DDS` and `Tools/ros2`.
- `ruff` linting is also enforced via pre-commit.
- `isort` with `profile="black"` for import ordering.

---

## Commit Messages

Format: `Subsystem: short description`

Rules:

- First line **must** contain a `:` subsystem prefix. Keep under ~72 chars.
- **No merge commits** — always rebase. **No `fixup!` commits** — squash first.
- One logical change per commit. Split unrelated changes.
- **Every file in a commit must belong to the declared subsystem.**

The definitive list of allowed prefixes: `Tools/scripts/allowed_subsystems.py`. A prefix is allowed if it is a `libraries/` directory name, or one of the curated extras (vehicle names like `Plane:`, `Copter:`, plus `Tools`, `autotest`, `waf`, `hwdef`, `modules`, etc.).

### Multi-subsystem tooling

```sh
# Split a multi-subsystem commit into one commit per subsystem
git subsystems-split                          # HEAD only
git subsystems-split --branch [BASE]          # entire branch

# Dry-run first
git subsystems-split -n

# Verify before pushing (this is what CI runs)
Tools/scripts/check_branch_conventions.py
```

---

## Parameter Documentation

Parameters use inline C++ annotations above `AP_GROUPINFO`:

```cpp
// @Param: ENABLE
// @DisplayName: Terrain data enable
// @Description: enable terrain data.
// @Values: 0:Disable,1:Enable
// @User: Advanced
AP_GROUPINFO_FLAGS("ENABLE", 0, AP_Terrain, enable, 1, AP_PARAM_FLAG_ENABLE),
```

Available: `@Param:`, `@DisplayName:`, `@Description:`, `@Values:`, `@Bitmask:`, `@Range:`, `@Units:`, `@Increment:`, `@User:` (Standard/Advanced), `@RebootRequired:`, `@Vehicles:`.

Parameter fullname max length is **16 characters**.

---

## Repository Layout

```text
ArduCopter/ ArduPlane/ ArduSub/ Rover/ AntennaTracker/ Blimp/   # Vehicle code
Tools/AP_Periph/                                                  # CAN peripheral firmware
libraries/                                                        # Shared libraries (bulk of code)
  AP_<Name>/          # Core libraries (AP_GPS, AP_Baro, ...)
  AC_<Name>/          # Controls (AC_PID, AC_WPNav, AC_AttitudeControl, ...)
  AR_<Name>/          # Rover-specific (AR_Motors, AR_WPNav)
  AP_HAL*/            # Hardware abstraction layers
  GCS_MAVLink/        # MAVLink GCS interface
  SITL/               # SITL simulation physics
Tools/
  autotest/           # SITL integration test framework
  scripts/            # Build/CI scripts
  CodeStyle/          # astyle config
  ardupilotwaf/       # Waf build system extensions
  gittools/           # Git helpers (subsystem-split, etc.)
modules/              # Git submodules (ChibiOS, mavlink, gtest...)
```

Each library typically has: main `.h`/`.cpp`, `*_Backend.*` interface, `*_config.h` for compile-time flags, optional `tests/` and `examples/`.
Each vehicle has: main class inheriting `AP_Vehicle`, `mode_*.cpp`, `Parameters.cpp`/`.h`, `GCS_*.cpp`/`.h`, `wscript`.

---

## Critical Constraints

- **Safety-critical**: Never fabricate test results, APIs, parameters, MAVLink messages, or hardware interfaces. Verify against actual source.
- **Don't guess at control loops, failsafe logic, or sensor fusion** — flag for human review.
- **Never modify submodules** (`modules/`) — managed upstream.
- **Never change parameter indices** (`AP_GROUPINFO` index numbers) — breaks stored configs.
- **Respect `#if AP_<FEATURE>_ENABLED` guards** — don't remove them.
- **No platform-specific code in shared libraries** — use HAL abstraction.
- **No unnecessary dependencies** — every byte of RAM/flash matters on embedded hardware.
- **Minimal, targeted changes** — no speculative large refactors.
- **Disclose AI assistance** in PR descriptions.

---

## CI Checks (on PRs to ArduPilot/ardupilot)

- SITL tests per vehicle (Copter, Plane, Rover, Sub, Tracker, Blimp, Periph)
- C++ unit tests (GCC + Clang)
- ChibiOS board builds
- astyle formatting
- flake8 / ruff Python linting
- Commit message format and subsystem prefix allow-list
- Binary size tracking
- Pre-commit hooks (line endings, codespell, XML/YAML)
- Markdown linting

You can test by opening a PR to your own fork first.
