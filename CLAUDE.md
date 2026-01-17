# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

OrcaSlicer is an open-source 3D slicer application forked from Bambu Studio, built using C++ with wxWidgets for the GUI and CMake as the build system. The project uses a modular architecture with separate libraries for core slicing functionality, GUI components, and platform-specific code.

## Quick Reference

Common commands for daily development:

```bash
# Build on your platform
./build_release_vs2022.bat           # Windows
./build_release_macos.sh             # macOS
./build_linux.sh -dsi                # Linux

# Run tests
cd build && ctest --output-on-failure
cd build && ./tests/libslic3r/libslic3r_tests --order rand --warn NoAssertions

# Format code
clang-format -i <file>

# Check profiles
python scripts/orca_extra_profile_check.py

# Update translations
./scripts/run_gettext.sh             # Linux/macOS
scripts\run_gettext.bat              # Windows
```

## Build Commands

### Building on Windows
```bash
# Build everything
build_release_vs2022.bat

# Build with debug symbols
build_release_vs2022.bat debug

# Build only dependencies
build_release_vs2022.bat deps

# Build only slicer (after deps are built)
build_release_vs2022.bat slicer


```

### Building on macOS
```bash
# Build everything (dependencies and slicer)
./build_release_macos.sh

# Build only dependencies
./build_release_macos.sh -d

# Build only slicer (after deps are built)
./build_release_macos.sh -s

# Use Ninja generator for faster builds
./build_release_macos.sh -x

# Build for specific architecture
./build_release_macos.sh -a arm64    # or x86_64 or universal

# Build for specific macOS version target
./build_release_macos.sh -t 11.3
```

### Building on Linux
```bash
# First time setup - install system dependencies
./build_linux.sh -u

# Build dependencies and slicer
./build_linux.sh -dsi

# Build everything (alternative)
./build_linux.sh -dsi

# Individual options:
./build_linux.sh -d    # dependencies only
./build_linux.sh -s    # slicer only  
./build_linux.sh -i    # build AppImage

# Performance and debug options:
./build_linux.sh -j N  # limit to N cores
./build_linux.sh -1    # single core build
./build_linux.sh -b    # Debug build
./build_linux.sh -e    # RelWithDebInfo build
./build_linux.sh -c    # clean build
./build_linux.sh -r    # skip RAM/disk checks
./build_linux.sh -l    # use Clang instead of GCC
```

### Build System
- Uses CMake with minimum version 3.13 (maximum 3.31.x on Windows)
- Primary build directory: `build/`
- Dependencies are built in `deps/build/`
- The build process is split into dependency building and main application building
- Windows builds use Visual Studio generators
- macOS builds use Xcode by default, Ninja with -x flag
- Linux builds use Ninja generator

### Testing
Tests are located in the `tests/` directory and use the Catch2 v2 testing framework. Test structure:
- `tests/libslic3r/` - Core library tests (21 test files)
  - Geometry processing, algorithms, file formats (STL, 3MF, AMF)
  - Polygon operations, clipper utilities, Voronoi diagrams
- `tests/fff_print/` - Fused Filament Fabrication tests (12 test files)
  - Slicing algorithms, G-code generation, print mechanics
  - Fill patterns, extrusion, support material
- `tests/sla_print/` - Stereolithography tests (4 test files)
  - SLA-specific printing algorithms, support generation
- `tests/libnest2d/` - 2D nesting algorithm tests
- `tests/slic3rutils/` - Utility function tests
- `tests/sandboxes/` - Experimental/sandbox test code

Run all tests after building:
```bash
cd build && ctest
```

Run tests with verbose output:
```bash
cd build && ctest --output-on-failure
```

Run individual test suites (recommended for development):
```bash
# From build directory - always use random order for reliable tests
cd build && ./tests/libslic3r/libslic3r_tests --order rand --warn NoAssertions
cd build && ./tests/fff_print/fff_print_tests --order rand --warn NoAssertions

# Filter by tags or patterns
cd build && ./tests/libslic3r/libslic3r_tests "[Geometry]" --order rand
cd build && ./tests/libslic3r/libslic3r_tests "*polygon*" --order rand
```

**Important**: See `tests/CLAUDE.md` for comprehensive testing guide including critical rules about thread safety, floating-point comparisons, and Catch2 best practices.

## Architecture

### Core Libraries
- **libslic3r/**: Core slicing engine and algorithms (platform-independent)
  - Main slicing logic, geometry processing, G-code generation
  - Key classes: Print, PrintObject, Layer, GCode, Config
  - Modular design with specialized subdirectories:
    - `GCode/` - G-code generation, cooling, pressure equalization, thumbnails
    - `Fill/` - Infill pattern implementations (gyroid, honeycomb, lightning, etc.)
    - `Support/` - Tree supports and traditional support generation
    - `Geometry/` - Advanced geometry operations, Voronoi diagrams, medial axis
    - `Format/` - File I/O for 3MF, AMF, STL, OBJ, STEP formats
    - `SLA/` - SLA-specific print processing and support generation
    - `Arachne/` - Advanced wall generation using skeletal trapezoidation

- **src/slic3r/**: Main application framework and GUI
  - GUI application built with wxWidgets
  - Integration between libslic3r core and user interface
  - Main GUI components in `src/slic3r/GUI/`:
    - 3D viewport and bed visualization (3DScene.cpp, 3DBed.cpp)
    - AMS (Automated Material System) widgets and controls
    - Plater (main workspace for object manipulation)
    - Print and material settings dialogs
    - Network printing interface (Bambu Lab, Klipper, OctoPrint integration)
  - Configuration management in `src/slic3r/Config/`
  - Platform utilities in `src/slic3r/Utils/`

### Key Algorithmic Components
- **Arachne Wall Generation**: Variable-width perimeter generation using skeletal trapezoidation
- **Tree Supports**: Organic support generation algorithm  
- **Lightning Infill**: Sparse infill optimization for internal structures
- **Adaptive Slicing**: Variable layer height based on geometry
- **Multi-material**: Multi-extruder and soluble support processing
- **G-code Post-processing**: Cooling, fan control, pressure advance, conflict checking

### File Format Support
- **3MF/BBS_3MF**: Native format with extensions for multi-material and metadata
- **STL**: Standard tessellation language for 3D models
- **AMF**: Additive Manufacturing Format with color/material support  
- **OBJ**: Wavefront OBJ with material definitions
- **STEP**: CAD format support for precise geometry
- **G-code**: Output format with extensive post-processing capabilities

### External Dependencies
- **Clipper2**: Advanced 2D polygon clipping and offsetting
- **libigl**: Computational geometry library for mesh operations
- **TBB**: Intel Threading Building Blocks for parallelization
- **wxWidgets**: Cross-platform GUI framework
- **OpenGL**: 3D graphics rendering and visualization
- **CGAL**: Computational Geometry Algorithms Library (selective use)
- **OpenVDB**: Volumetric data structures for advanced operations
- **Eigen**: Linear algebra library for mathematical operations

## File Organization

### Resources and Configuration
- `resources/profiles/` - Printer and material profiles organized by manufacturer
- `resources/printers/` - Printer-specific configurations and G-code templates  
- `resources/images/` - UI icons, logos, calibration images
- `resources/calib/` - Calibration test patterns and data
- `resources/handy_models/` - Built-in test models (benchy, calibration cubes)

### Internationalization and Localization  
- `localization/i18n/` - Source translation files (.pot, .po)
- `resources/i18n/` - Runtime language resources
- Translation managed via `scripts/run_gettext.sh` / `scripts/run_gettext.bat`

### Platform-Specific Code
- `src/libslic3r/Platform.cpp` - Platform abstractions and utilities
- `src/libslic3r/MacUtils.mm` - macOS-specific utilities (Objective-C++)
- Windows-specific build scripts and configurations
- Linux distribution support scripts in `scripts/linux.d/`

### Build and Development Tools
- `cmake/modules/` - Custom CMake find modules and utilities
- `scripts/` - Python utilities for profile generation and validation  
- `tools/` - Windows build tools (gettext utilities)
- `deps/` - External dependency build configurations

## Development Workflow

### Code Style and Standards
- **C++17 standard** with selective C++20 features
- **Naming conventions**: PascalCase for classes, snake_case for functions/variables
- **Header guards**: Use `#pragma once`
- **Memory management**: Prefer smart pointers, RAII patterns
- **Thread safety**: Use TBB for parallelization, be mindful of shared state
- **Code formatting**: Uses `.clang-format` with 4-space indents, 140-column limit
  - Run `clang-format -i <file>` before committing
  - CMake target available when LLVM tools are on PATH: `make clang-format`
  - Key settings: aligned assignments/declarations, custom brace wrapping, pointer alignment left

### Common Development Tasks

#### Adding New Print Settings
1. Define setting in `PrintConfig.cpp` with proper bounds and defaults
2. Add UI controls in appropriate GUI components  
3. Update serialization in config save/load
4. Add tooltips and help text for user guidance
5. Test with different printer profiles

#### Modifying Slicing Algorithms  
1. Core algorithms live in `libslic3r/` subdirectories
2. Performance-critical code should be profiled and optimized
3. Consider multi-threading implications (TBB integration)
4. Validate changes don't break existing profiles
5. Add regression tests where appropriate

#### GUI Development
1. GUI code resides in `src/slic3r/GUI/` (not visible in current tree)
2. Use existing wxWidgets patterns and custom controls
3. Support both light and dark themes
4. Consider DPI scaling on high-resolution displays
5. Maintain cross-platform compatibility

#### Adding Printer Support
1. Create JSON profile in `resources/profiles/[manufacturer].json`
2. Add printer-specific start/end G-code templates
3. Configure build volume, capabilities, and material compatibility
4. Test thoroughly with actual hardware when possible
5. Follow existing profile structure and naming conventions

### Dependencies and Build System
- **CMake-based** with separate dependency building phase
- **Dependencies** built once in `deps/build/`, then linked to main application
- **Cross-platform** considerations important for all changes
- **Resource files** embedded at build time, platform-specific handling
- **Version management**: Version defined in `version.inc` (e.g., `2.3.2-dev`)
  - Git commit hash automatically embedded during CI builds
  - Separate SLIC3R_VERSION for compatibility tracking

### CI/CD and Automation
- **GitHub Actions** workflows in `.github/workflows/`:
  - `build_all.yml` - Main build workflow for all platforms (triggered on main/release branches)
  - `build_deps.yml` - Separate dependency building
  - `check_profiles.yml` - Validates printer/filament profiles
  - `check_locale.yml` - Checks translation files
  - `doxygen-docs.yml` - Generates code documentation
- **Automated checks**: Profile validation, locale completeness, shell script linting
- **Nightly builds**: Scheduled daily builds at 1 AM Singapore time (UTC+8)
- **Manual dispatch**: Workflows support manual triggering with options

### Development Scripts and Utilities
- **Profile management**: `scripts/generate_presets_vendors.py` - Generates vendor preset files
- **Profile validation**: `scripts/orca_extra_profile_check.py` - Validates profile JSON syntax and structure
- **Filament library**: `scripts/orca_filament_lib.py` - Filament profile utilities
- **Image optimization**: `scripts/optimize_cover_images.py` - Compresses and optimizes cover images
- **Localization**:
  - `scripts/run_gettext.sh` / `scripts/run_gettext.bat` - Extracts translatable strings
  - `scripts/HintsToPot.py` - Processes UI hints for translation
- **Packaging**: `scripts/pack_profiles.sh` - Packages profiles for distribution
- **Docker support**: `scripts/DockerBuild.sh` and `scripts/DockerRun.sh` - Reproducible container builds

### Performance Considerations
- **Slicing algorithms** are CPU-intensive, profile before optimizing
- **Memory usage** can be substantial with complex models
- **Multi-threading** extensively used via TBB
- **File I/O** optimized for large 3MF files with embedded textures
- **Real-time preview** requires efficient mesh processing

## Important Development Notes

### Codebase Navigation
- Use search tools extensively - codebase has 500k+ lines
- Key entry points: `src/OrcaSlicer.cpp` for application startup
- Core slicing: `libslic3r/Print.cpp` orchestrates the slicing pipeline
- Configuration: `PrintConfig.cpp` defines all print/printer/material settings

### Compatibility and Stability
- **Backward compatibility** maintained for project files and profiles
- **Cross-platform** support essential (Windows/macOS/Linux)  
- **File format** changes require careful version handling
- **Profile migrations** needed when settings change significantly

### Quality and Testing
- **Regression testing** important due to algorithm complexity
- **Performance benchmarks** help catch performance regressions
- **Memory leak** detection important for long-running GUI application
- **Cross-platform** testing required before releases

## Known Build Issues

### Windows - "v145 build tools cannot be found" Error (VS2022)

**Problem**: Build fails with error `The build tools for v145 (Platform Toolset = 'v145') cannot be found.`

**Important**: This error message is misleading! The "v145" toolset doesn't actually exist in standard Visual Studio releases. The real issue is an incomplete or missing MSVC toolset installation.

**Root Cause**:
Multiple MSVC toolsets can be installed side-by-side in Visual Studio 2022, typically in:
- `C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.42.xxxxx\`
- `C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.44.xxxxx\`

The problem occurs when:
1. An older toolset (e.g., 14.42) is installed but **missing x64 runtime libraries** (MSVCRTD.lib, LIBCMT.lib, etc.)
2. CMake generates projects using the incomplete toolset
3. Nested dependency builds (OpenCV, Boost, etc.) fail with linker errors like:
   - `LINK : fatal error LNK1104: cannot open file 'MSVCRTD.lib'`
   - `LINK : fatal error LNK1104: cannot open file 'LIBCMT.lib'`

**Verification**:
Check which toolsets are installed and if they have x64 libraries:
```powershell
# List installed toolsets
ls "C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\"

# Check if x64 libraries exist for each version
ls "C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.44.35207\lib\x64\"
```

If a toolset directory lacks an `x64` subdirectory under `lib\`, it's incomplete.

**Solutions**:

*Option 1: Remove Incomplete Toolsets (Recommended)*
1. Open **Visual Studio Installer**
2. Click **"Modify"** on Visual Studio 2022
3. Go to **"Individual Components"** tab
4. Scroll to **"Compilers, build tools, and runtimes"**
5. **Uncheck** any incomplete toolset versions (e.g., v14.42)
6. Keep only the latest complete toolset checked
7. Click **"Modify"** to apply changes

*Option 2: Install Missing Components*
1. In Visual Studio Installer → Individual Components
2. Ensure these are checked:
   - **MSVC v143 - VS 2022 C++ x64/x86 build tools (Latest)**
   - **Windows 10 SDK** (any recent version)
   - **C++ CMake tools for Windows**

*Option 3: Force Specific Toolset Version*
When configuring CMake, explicitly specify the working toolset:
```bash
cmake .. -G "Visual Studio 17 2022" -A x64 -T v143,version=14.44
```

**Installation via PowerShell** (requires Administrator):
```powershell
& "C:\Program Files (x86)\Microsoft Visual Studio\Installer\setup.exe" modify `
  --installPath "C:\Program Files\Microsoft Visual Studio\2022\Community" `
  --add Microsoft.VisualStudio.Workload.NativeDesktop `
  --includeRecommended `
  --quiet --norestart
```

**After Fixing**:
Clean all build artifacts and rebuild:
```bash
cd C:\Users\<your-user>\source\repos\Stan-Elston\OrcaSlicer
rm -rf deps/build build
./build_release_vs2022.bat
```

### Windows - OpenCV Precompiled Header Conflict (VS2026/VS2022)

**Problem**: Build fails with error `error C2653: 'cv': is not a class or namespace name` in ObjColorUtils.cpp/hpp

**Root Cause**:
- OrcaSlicer uses precompiled headers (PCH) that are force-included before all other headers
- Two PCH files exist: `src/libslic3r/pchheader.hpp` and `src/slic3r/pchheader.hpp`
- Neither PCH originally includes OpenCV headers (`opencv2/opencv.hpp`)
- ObjColorUtils.hpp uses OpenCV types (cv::Mat, cv::Vec3f, etc.) but when included by other files, the PCH has already been processed without OpenCV definitions

**Files Affected**:
- `src/libslic3r/ObjColorUtils.hpp` - Only file that uses OpenCV in libslic3r
- `src/libslic3r/ObjColorUtils.cpp`
- `src/slic3r/GUI/ObjColorDialog.cpp` - GUI file that includes ObjColorUtils.hpp
- `src/slic3r/GUI/Plater.cpp` - GUI file that includes ObjColorUtils.hpp

**Solutions**:

*Option 1: Disable Precompiled Headers (Recommended Workaround)*
```bash
cmake .. -G "Visual Studio 18 2026" -A x64 -DORCA_TOOLS=ON -DCMAKE_BUILD_TYPE=Release -DSLIC3R_PCH=OFF
```
This makes compilation slower but ensures all headers are processed correctly.

*Option 2: Add OpenCV to Both Precompiled Headers (Not Working Reliably)*
Edit both `src/libslic3r/pchheader.hpp` and `src/slic3r/pchheader.hpp` to add at the end:
```cpp
// OpenCV - must be at the end to avoid conflicts
#include <opencv2/opencv.hpp>
```
Note: This approach has proven unreliable with Visual Studio's PCH caching. Even after clean rebuilds, the PCH may not be regenerated correctly.

**Additional Notes**:
- Boost version was also updated from 1.83.0 to 1.84.0 in CMakeLists.txt:579
- Visual Studio 2026 (version 18) is very new and may have additional quirks
- The build_release_vs.bat script auto-detects VS version correctly
