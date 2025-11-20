# Firmware Style Guide

**Version:** 1.0
**Team Size:** 2-person embedded systems team
**Applies To:** Arduino, ESP32, RP2040, Teensy, and general embedded C++ firmware

---

# 1. Project Structure

### 1.1 Directory Layout

* `src/` — All firmware source files (.cpp, .h)
* `hardware/` — Wiring diagrams, schematics, power notes
* `docs/` — maintenance notes, troubleshooting guides

### 1.2 File Placement

* All custom classes (`*.h` + `*.cpp`) reside directly in `src/` (Arduino IDE–friendly)
* No per‑class subfolders

---

# 2. Naming Conventions

### 2.1 Variables

* Local variables: `lowerCamelCase`
* Member variables: `lowerCamelCase_` (trailing underscore)
* Global variables: avoid unless necessary; same style as local

### 2.2 Constants & Enums

* Constants: `ALL_CAPS_WITH_UNDERSCORES`
* Enum values: `kPascalCase`
* Enum types: `PascalCase`

### 2.3 Files & Classes

* Classes: `PascalCase`
* Filenames: match class name exactly (e.g., `AudioPlayer.h` / `AudioPlayer.cpp`)

---

# 3. File Headers & Documentation

### 3.1 File Header Template

Every `.h`, `.cpp`, and `.ino` begins with:

```
/**
 * File: <filename>
 * Project: <project title>
 * Author: <name>
 * Date: <yyyy-mm-dd>
 * Description: <short description of this file>
 * Notes: <dependencies, pin usage, modules, quirks, etc.>
 */
```

### 3.2 Function Documentation

Use short but descriptive Doxygen-style comments:

```
/// Brief description of what the function does.
/// More detailed explanation if needed.
/// @param value Description of parameter
/// @return Description of return value
```

Short one-liners are acceptable for very small/obvious functions.

### 3.3 Inline Comments

Inline comments are allowed and encouraged when they:

* Explain *why*, not obvious *what*
* Describe hardware/software quirks, timing considerations, or design rationale

No strict density rule—use judgment. Prioritize clarity.

---

# 4. Code Structure & OOP Practices

* Classes follow single-responsibility design
* Public API very minimal, internal helpers go in `private:`
* Avoid dynamic allocation unless necessary
* Prefer fixed-size buffers and stack usage in microcontroller code
* Functions ideally remain under ~40 lines, but no enforced limit (but prioritize conciseness)

---

# 5. Formatting Rules

### 5.1 Indentation & Spacing

* Indentation: **2 spaces** (matches clang autoformat)
* Line length: follow clang format default (typically 100–120 chars)
* Brace style:

```
if (condition) {
  ...
}
```

### 5.2 Switch Statements

Each `case` should include at least one line of explanation when logic is non-trivial.

---

# 6. Hardware Documentation Requirements

### 6.1 Power Notes

Every project must document:

* Power calculations
* Power sources and voltage rails
* Startup sequence notes
* Known quirks on boot (brownouts, timing issues)

### 6.2 Wiring Diagrams

Include `hardware/wiring.pdf` or `.png` checked into the repo.

### 6.3 Maintenance Notes

Each repo includes `docs/MAINTENANCE.md` covering:

* Known failure modes
* Replacement part numbers or links
* Reset procedures
* Calibration steps (if any)

---

# 7. README Template

A project README includes:

1. **Overview:** 5–10 sentence summary
2. **Hardware List:** microcontrollers, sensors, power supplies
3. **Wiring Diagram or Link**
4. **Software Architecture:** key classes and data flow
5. **Setup Instructions:** build + flashing
6. **Images/Photos** allowed and encouraged
7. **Troubleshooting & Known Issues**

---

# 8. Versioning & Branching

* `main` = stable production firmware
* Feature branches for all new work
* Descriptive tags with version numbering preferred (e.g., `v1.2-mkrzero-i2s`)
* Tag all production-ready releases

---

# 9. Summary

This style guide aims to keep firmware readable, maintainable, and consistent across a small embedded team. Clarity, documentation, and predictable structure take priority over strictness.
