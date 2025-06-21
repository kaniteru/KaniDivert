# Changelog

All notable changes to this project will be documented in this file.

---

## v250621

### Added
- CMake-based build system
- Support for both static and shared library builds

### Changed sources
- windivert.h
- windivert.c
```
Added conditional logic for the `WINDIVERTSTATIC` macro to support static linking.
```