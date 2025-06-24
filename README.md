# KaniDivert

KaniDivert is a personal fork of [WinDivert](https://github.com/basil00/WinDivert).

It is not affiliated with or endorsed by the original WinDivert project.

---

## Features

- CMake-based build system
- Supports kernel driver and both static and shared library builds
- Includes fixes and improvements from other forked repositories

---

## Supported platforms

- **SYS**
  - MSVC
    - x64: tested and working
    - x86: _not tested_

- **DLL**
  - MSVC
    - x64: tested and working
    - x86: _not tested_
  - MinGW
    - x64: _not tested_
    - x86: _not tested_

---

## Installation

### Using 'FetchContent' in CMake

1. Include and fetch the repository
   ```cmake
   include(FetchContent)
   FetchContent_Declare(
           kanidivert
           GIT_REPOSITORY https://github.com/kaniteru/KaniDivert
           GIT_TAG        master # You can replace 'master' with a specific tag or branch if needed.
   )
   FetchContent_MakeAvailable(kanidivert)
   ```

2. Link to your project
   ```cmake
   target_link_libraries(your_project PRIVATE KaniDivert::shared)
   ```
   or
   ```cmake
   target_link_libraries(your_project PRIVATE KaniDivert::static)
   ```

3. (Optional) Download official WinDivert driver files
   ```cmake
   # Downloads the official .sys driver files on post build
   download_official_sys(your_project "${download_path}")
   ```

### Using prebuilt binaries
1. Download prebuilt binaries from the [Releases](https://github.com/kaniteru/KaniDivert/releases) page.
2. If you're using the static library, make sure to define **WINDIVERTSTATIC** in your compiler flags.

---

## License

KaniDivert follows the same dual-license terms as the original [WinDivert](https://reqrypt.org/windivert.html):

- GNU Lesser General Public License (LGPL) v3, or
- GNU General Public License (GPL) v2

This project also uses FindWDK (BSD 3-Clause License) for build system integration.

Please refer to the [LICENSE](./LICENSE) file for full details.