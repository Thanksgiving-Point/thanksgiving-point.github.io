---
layout: styleguide
permalink: /styleguide.html
---

# C++/Firmware Style Guide

**Version 2.0** (2026-07-31)

A lightweight, practical style guide for the embedded firmware used across our exhibits.
Designed for a small team: emphasis on clarity, maintainability, and embedded best practices.

It applies to new code and to files you're already editing. Don't retrofit a
working exhibit just to match it.

## 1. Project & File Organization

### 1.1 Repository layout

One repo per exhibit interactive. One directory per microcontroller **sketch**, named after
the sketch's _role_, not its chip (projects routinely end up with two boards of
the same type).

```
exhibit-name/
├── .clang-format            # formatting, see §2.1
├── .clangd                  # editor/LSP support only, not the build
├── .gitignore
├── README.md                # see §8.4
├── docs/                    # schematics, diagrams, photos
├── sensor-controller/       # a sketch, named by role
│   ├── sensor-controller.ino
│   ├── sketch.yaml
│   └── src/
│       ├── App.h            # coordinates subsystems (§1.2)
│       ├── App.cpp
│       ├── Config.h
│       ├── Debug.h
│       └── ...
└── led-driver/              # another mcu sketch
```

The sketch folder and the `.ino` file must share a name - `arduino-cli`
requires it.

### 1.2 The `.ino` sketch stays thin

Real logic lives in `src/` classes. The `.ino` handles bring-up in the `setup()` function - it starts
`Serial`, starts buses (`Wire.h`), and hands off the rest in the `loop()` function.

If a sketch coordinates two or more stateful subsystems, or runs a state
machine spanning them, consider a central `App` class to own that coordination.
Simpler sketches don't need it - `setup()`/`loop()` can call into `src/` classes
directly.

> See the `piston-dynamics/` sketch in [`piston-interactive`](https://github.com/iwonder77/piston-interactive) for an
> example of an `App` class.

## 2. Formatting & Naming

### 2.1 Formatting

Formatting is not a matter of taste here - `clang-format` decides. Copy this
`.clang-format` into every repo next to `.clangd`, and format on save.

```yaml
BasedOnStyle: LLVM
```

LLVM's defaults are what we already write: two-space indent, 80-column limit,
`Type &name` reference placement. Nothing else in this guide has an opinion on
braces, indentation, or line length, because nothing else needs one.

To fold in a file that predates the config, reformat it in its own `style:`
commit (§9.1) so the diff stays reviewable.

### 2.2 Names

| Kind                        | Convention                                                         | Example                           |
| --------------------------- | ------------------------------------------------------------------ | --------------------------------- |
| Local variables, parameters | `snake_case`                                                       | `elapsed_time`, `raw_reading`     |
| Member variables            | `snake_case_` (trailing underscore)                                | `rpm_`, `error_count_`            |
| Functions & methods         | `camelCase`                                                        | `initPWM()`, `decayRPM()`         |
| Classes, structs, types     | `PascalCase`                                                       | `EngineModel`, `LedOutput`        |
| Enum types                  | `PascalCase`, always `enum class` with an explicit underlying type | `enum class PistonSize : uint8_t` |
| Enum values                 | `UPPER_SNAKE_CASE`                                                 | `SMALL`, `ERROR_RECOVERY`         |
| Constants (`constexpr`)     | `UPPER_SNAKE_CASE`                                                 | `MAX_VALID_MM`                    |
| Namespaces                  | `snake_case`                                                       | `config`, `motion_detection`      |
| Macros                      | `UPPER_SNAKE_CASE`                                                 | `DEBUG_PRINTLN`                   |
| Files                       | Match the primary class                                            | `EngineModel.h`                   |

### 2.3 Units in names

Any constant or variable carrying a physical quantity names its unit. This is
the cheapest bug prevention in the guide - a `_MS` next to a `_US` catches
mistakes that types can't.

Examples: `TIMEOUT_MS`, `PULSE_US`, `MAX_VALID_MM`, `PWM_FREQ_HZ`, `TORQUE_LED_PIN`.

## 3. Class Design

### 3.1 Header-only vs. header + implementation

**Split into `.h`/`.cpp`** when either is true:

- The class touches a peripheral - GPIO, I2C, SPI, PWM, ADC, interrupts
  (`millis()` and `micros()` don't count)
- The implementation is long or has non-trivial private helpers

**Otherwise keep it header-only:** small, pure-logic classes like filters,
timers, debouncers, and health monitors. Define methods inside the class body
so they're implicitly `inline` - keep those definitions to trivial getters,
single-line logic, and small math helpers.

Anything longer goes out-of-class as `inline` further down the same header - a
debouncer's `update()` doesn't have to squeeze into the declaration to stay
header-only. When in doubt, split.

### 3.2 Other class design notes

- Use `#pragma once` at the top of every header.
- **Static methods:** use `static` if the method does not access instance state
  and conceptually belongs to the class. A class with no members at all is a
  sign the methods should be `static`.
- Keep single responsibility per class.
- Mark every method that doesn't mutate state `const`.
- Initialize members at the point of declaration, not in the constructor body.
- **Constructors don't touch hardware.** File-scope objects are constructed
  before `setup()` runs, before the Arduino core has initialized - no
  `pinMode()`, no `Wire`, no `Serial` in a constructor. Hardware setup belongs
  in `init()` (§4), which is why that method exists.

### 3.3 Hardware-owning classes are non-copyable

Any class that owns a peripheral, a buffer, or an ISR registration must delete
copy construction and assignment. Copying a driver produces two objects that
each believe they exclusively own one piece of hardware, and the resulting bugs
are subtle and intermittent.

```cpp
class ToFSensor {
public:
  ToFSensor() = default;
  ToFSensor(const ToFSensor &) = delete;
  ToFSensor &operator=(const ToFSensor &) = delete;
  // ...
};
```

To re-initialize hardware, call `init()` again on the existing object. Never
construct a replacement and assign over the original.

## 4. Error Handling & Robustness

- Track errors with counters / boolean flags and recovery timers (use
  `millis()`).
- Hardware setup goes in an `init()` method - one verb, for first bring-up and
  re-initialization alike. It returns `bool`, and **callers must check the
  result**. On `false`, record the failure and enter the recovery state (§4.1) -
  never carry on as though it succeeded.
- Saturate error counters so they can't wrap: `if (count_ < UINT8_MAX) ++count_;`

### 4.1 Exhibits never halt

An exhibit that hangs stays dark until someone notices. An exhibit that retries
usually fixes itself before opening. Never `while (1) {}`, never block forever
waiting on a peripheral, never let a failed sensor stop the main loop.

Failure handling is a state, not a stop: enter a recovery state, retry on a
timer, and keep looping. Degrade to a sensible default output rather than
freezing the last value or going black.

### 4.2 Feed a watchdog

§4.1 is a rule you can follow perfectly and still hang, because the blocking
happens in code you don't own - a wedged I2C slave, a library call with no
timeout. Enable the chip's watchdog (`esp_task_wdt_*` on ESP32, `wdt_enable()`
on AVR) and feed it once per `loop()`, with the timeout as a named constant in
`Config.h`. A reboot is a poor recovery, but it beats a dark exhibit.

Feed it only from the main loop - never from a timer or an ISR. The point is to
prove the loop is still turning, not that the chip still has a clock.

Set explicit bus timeouts for the same reason:
`Wire.setTimeOut(config::I2C_TIMEOUT_MS)`.

## 5. `Config.h` - Global Constants, Namespaces, and Utilities

- Place all tunables in `config::` as `constexpr`, in one `Config.h` per node.
- Avoid polluting the global namespace.
- Name constants with units where relevant (§2.3).
- Group related constants under a `// ----- SECTION -----` comment.
- Very small pure helpers may live in `util::` or `config::` as `static inline`
  functions - `clampf(...)` and the like.
- No magic numbers in logic files. If a literal has meaning, it's a constant.
- Do **not** place medium/large logic or hardware code in utility headers.

### 5.1 Derived values are constants too

Anything computed from another constant belongs in `Config.h` with a name, not
inline at the call site. An unnamed expression in a loop is a value nobody can
search for.

```cpp
// Good
constexpr uint32_t SENSOR_POLL_MS = TIMING_BUDGET_US / 1000;
if (now - last_read_ms_ >= config::SENSOR_POLL_MS) { ... }

// Avoid
if (now - last_read_ms_ >= config::TIMING_BUDGET_US / 1000) { ... }
```

### 5.2 Per-installation variants

When one codebase ships to several physical units with different constants,
mark the variant loudly at the top of `Config.h` and keep it to a _single_
switch.

```cpp
// ===== SET PER INSTALLATION - see README "Deployment" =====
constexpr PistonSize DISPLAY_TYPE = PistonSize::SMALL;
```

Tag each deployment with its variant (§9.2) so the repo records what's on
which unit.

## 6. Embedded Best Practices

- No dynamic allocation - no `new`, no `malloc`, no Arduino `String`. Use
  fixed-size arrays and stack allocation, with sizes from `constexpr` values or
  template parameters.
- No blocking calls in the main loop - no `delay()`, use non-blocking timing.
  `delay()` in `setup()` for hardware settling is fine.
- Use `<stdint.h>` types with explicit width (`uint8_t`, `uint32_t`), not `int`
  or `unsigned long` - their widths vary by platform. `uint32_t` for
  timestamps, `size_t` for sizes and indices.
- Use `float`, not `double` - most embedded FPUs are single-precision, so
  `double` math is emulated in software. Suffix literals with `f` (`0.5f`) and
  use the `f` variants of math functions (`fabsf`, `sqrtf`, `sinf`); an
  unsuffixed literal or a `double` function silently promotes the whole
  expression.

### 6.1 Timing must be rollover-safe

`millis()` wraps about every 49.7 days, `micros()` every 71 minutes. Always
**subtract** timestamps; never add to one and compare.

```cpp
// Good - correct across rollover
if (now - last_update_ms_ >= config::UPDATE_PERIOD_MS) { ... }

// Broken - fails once the counter wraps
if (now > last_update_ms_ + config::UPDATE_PERIOD_MS) { ... }
```

Capture `millis()` once at the top of a function and reuse it, so every
comparison in that pass shares one time base.

### 6.2 Interrupts

- Keep ISRs minimal (set a flag, record a timestamp); defer more complex work to the main loop or tasks.
- On ESP32, mark ISRs `IRAM_ATTR`.
- Every variable shared between an ISR and the main loop is `volatile`.
- In an ISR: no `Serial`, no `delay()`, no allocation, no floating-point math.
- Snapshot shared state into locals inside a critical section before using it -
  a multi-byte `volatile` read can tear.

```cpp
if (new_data_available) {
  noInterrupts();
  uint32_t safe_pulse_width = pulse_width;
  uint32_t safe_period = period_length;
  new_data_available = false;
  interrupts();
  // ... work on the local copies
}
```

## 7. Debug & Logging

Every project carries the same `Debug.h` so log statements compile out of a
release build. Copy it verbatim; if it needs to change, change it everywhere.

```cpp
#pragma once
/**
 * Debug.h
 *
 * Debug logging macros. Set DEBUG_LEVEL to 0 to compile out all output.
 */

#ifndef DEBUG_LEVEL
#define DEBUG_LEVEL 1
#endif

#if DEBUG_LEVEL >= 1
#define DEBUG_PRINT(...) Serial.print(__VA_ARGS__)
#define DEBUG_PRINTLN(...) Serial.println(__VA_ARGS__)
#define DEBUG_PRINTF(...) Serial.printf(__VA_ARGS__)
#else
#define DEBUG_PRINT(...)
#define DEBUG_PRINTLN(...)
#define DEBUG_PRINTF(...)
#endif
```

Use the macros in class code, never bare `Serial.print` - bare calls can't be
compiled out and slow a release build's main loop.

## 8. Documentation & Comments

### 8.1 File header

Every `.h` begins with:

```cpp
#pragma once
/**
 * FileName.h
 *
 * Small one paragraph max description with bullet points if necessary.
 */
```

The filename in the header must match the actual filename. `.cpp` files don't
repeat the block.

### 8.2 Function documentation

Short Doxygen-style header with brief description, params, and return:

```cpp
/**
 * @brief description of what the function does.
 * more detailed explanation if needed.
 *
 * @param <value> description of parameter
 * @return <value> description of return value
 */
bool camelCase(int param);
```

**The block goes on the declaration in the `.h`**, for header-only and split
classes alike.

Document every public method; include `@param` and `@return` where they add
something the signature doesn't. Private helpers get a one-line `//` comment or
nothing.

### 8.3 Inline comments

Explain _why_, not _what_ - hardware quirks, timing constraints, non-obvious
math or logic. Use `// NOTE:` for context a reader needs and `// TODO:` for
known gaps, rarely enough that both stay greppable.

### 8.4 README

Every exhibit repo's README covers, at minimum:

- **Overview** - what the exhibit does, from a visitor's point of view
- **Hardware** - boards, sensors, drivers, with links and a wiring diagram
- **Pinout** - a table, matching `Config.h`
- **Build & Flash** - exact `arduino-cli` commands, board package version, and
  pinned library versions
- **Deployment** - which variants exist, which unit gets which build (§5.2)
- **Maintenance** - symptoms, likely causes, how to recover. Write it for a
  technician who has never seen the code. This is the highest-value section in
  the repo.

Multi-class nodes also get an **Architecture** section: one short paragraph per
class, purpose and key methods.

## 9. Git & Repository Conventions

### 9.1 Commits

Short imperative subject, 72 characters or less. If it needs a comma or an "as
well as", the rest belongs in the body.

```
fix: remove blocking delay from the main loop

App polls the sensor on a timer now, so the 15 ms delay was capping the
update rate for no reason.
```

Prefixes keep `git log --oneline` skimmable: `feat`, `fix`, `docs`, `style`,
`chore`. A `style:` commit contains no functional change, so it's safe to skim
past when reviewing what actually shipped.

### 9.2 Tag every deployment

The repo must always be able to answer "what is running on that exhibit?" Tag
at the moment of flashing, one tag per unit when variants differ (§5.2).

```bash
git tag -a deploy/2026-07-31-small -m "flashed to small piston unit"
git push --tags
```

### 9.3 Repo hygiene

- `main` always compiles. Branch (`feat/…`, `fix/…`) when a change is risky or
  spans more than one sitting; otherwise commit straight to `main`.
- `.gitignore` build artifacts (`**/build`). Never ignore source or docs.
- `sketch.yaml` commits `default_fqbn` plus pinned `platforms:` and
  `libraries:` versions - that file, not the README prose, is what actually
  makes a build reproducible (§8.4). `default_port` differs per machine - edit
  it locally, don't commit the churn.
- Keep working copies out of cloud-sync folders (OneDrive, Dropbox). Syncing a
  `.git` directory corrupts it; GitHub is the backup.
- One-time setup: `git config --global user.email you@thanksgivingpoint.org`,
  `git config --global pull.rebase true`.
