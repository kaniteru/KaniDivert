# Changelog

All notable changes to this project will be documented in this file.

---

## v250624

### Added
- Support for kernel build
```
Added kernel sources into 'sys' from WinDivert repository
```
- Added 'download_official_sys' function in cmake
```
Download the WinDivert driver files (x64, x86) from WinDivert repository
```

### Changed
- Moved 'windivert_device.h' from 'src' to 'include'
- Moved dll source codes from 'src' to 'dll' 

### Changed sources
- sys/windivert.c
```
Fixed uninitialized pointer variables by adding NULL initialization.
```

---

## v250621

### Added
- CMake-based build system
- Support for both static and shared library builds

### Changed sources
- windivert.h
- src/windivert.c
```
Added conditional logic for the `WINDIVERTSTATIC` macro to support static linking.
```