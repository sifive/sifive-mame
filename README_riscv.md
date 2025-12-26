# Building MAME for RISC-V 64-bit

This guide describes how to cross-compile MAME for RISC-V 64-bit architecture from an x86_64 Linux host.

## Prerequisites

### Required Tools
- RISC-V 64-bit cross-compilation toolchain (e.g., `riscv64-unknown-linux-gnu-gcc`)
- Python 3
- Standard build tools: `make`, `gcc` (for host), `ar`, `rpm2cpio`, `cpio`

### RISC-V Sysroot Setup

A RISC-V sysroot contains the target system's libraries and headers needed for cross-compilation.

#### Obtaining a Sysroot

You can obtain a RISC-V sysroot in two ways:

1. **Use a prebuilt sysroot** from Freedom Tools or other RISC-V toolchain distributions
2. **Build your own sysroot** using FSFL (SiFive Freedom SDK For Linux)
   - Repository: https://github.com/sifiveinc/meta-sifive

Example sysroot location: `/scratch/work/mame/sysroot`

#### Required Libraries in Sysroot
- SDL2 development files (`libsdl2-dev`)
- SDL2_ttf development files (`libsdl2-ttf-dev`)

#### Recommended Libraries (for full features)
- X11 libraries (`libx11-dev`) - for X11 window management
- libXinerama (`libxinerama-dev`) - required for X11 support
- OpenGL libraries (`libgl-dev`, `libglu-dev`) - for OpenGL rendering
- ALSA libraries (`libasound-dev`) - for MIDI support
- PulseAudio libraries (`libpulse-dev`) - for PulseAudio audio output
- XInput libraries (`libxi-dev`) - for enhanced input device support
- Fontconfig (`libfontconfig-dev`) - for font support

**Note:** The build system will automatically detect available libraries in your sysroot and enable corresponding features.

#### Installing Libraries to Sysroot

##### SDL2_ttf (Required)
Download SDL2_ttf packages for RISC-V:
```bash
wget http://fedora.riscv.rocks/kojifiles/packages/SDL2_ttf/2.22.0/4.fc42/riscv64/SDL2_ttf-2.22.0-4.fc42.riscv64.rpm
wget http://fedora.riscv.rocks/kojifiles/packages/SDL2_ttf/2.22.0/4.fc42/riscv64/SDL2_ttf-devel-2.22.0-4.fc42.riscv64.rpm
```

Extract to your sysroot:
```bash
cd /path/to/sysroot
rpm2cpio SDL2_ttf-2.22.0-4.fc42.riscv64.rpm | cpio -idmv
rpm2cpio SDL2_ttf-devel-2.22.0-4.fc42.riscv64.rpm | cpio -idmv
```

Verify installation:
```bash
ls /path/to/sysroot/usr/include/SDL2/SDL_ttf.h
ls /path/to/sysroot/usr/lib64/pkgconfig/SDL2_ttf.pc
ls /path/to/sysroot/usr/lib64/libSDL2_ttf.so
```

##### libXinerama (Required for X11 support)
X11 window management requires libXinerama. Download and install it:

```bash
# Download libXinerama packages
wget https://riscv-kojipkgs.fedoraproject.org//packages/libXinerama/1.1.5/8.fc42/riscv64/libXinerama-1.1.5-8.fc42.riscv64.rpm
wget https://riscv-kojipkgs.fedoraproject.org//packages/libXinerama/1.1.5/8.fc42/riscv64/libXinerama-devel-1.1.5-8.fc42.riscv64.rpm

# Extract to sysroot
cd /path/to/sysroot
rpm2cpio libXinerama-1.1.5-8.fc42.riscv64.rpm | cpio -idmv
rpm2cpio libXinerama-devel-1.1.5-8.fc42.riscv64.rpm | cpio -idmv
```

Verify installation:
```bash
ls /path/to/sysroot/usr/lib64/libXinerama.so
ls /path/to/sysroot/usr/include/X11/extensions/Xinerama.h
```

## Source Code

### Download MAME Source
```bash
git clone https://github.com/mamedev/mame.git
cd mame
```

## Required Patches

The following files need to be modified to support RISC-V cross-compilation:

### 1. scripts/genie.lua

Add RISC-V specific compiler flags after line 1089:

```lua
if (_OPTIONS["PLATFORM"]=="riscv64") then
	buildoptions {
		"-Wno-cast-align",
		"-mcmodel=medany",
	}
end
```

### 2. scripts/src/3rdparty.lua

Add RISC-V specific flags to the `bx` project (around line 1332):

```lua
if (_OPTIONS["PLATFORM"]=="riscv64") then
	buildoptions {
		"-Wno-cast-align",
		"-mcmodel=medany",
		"-fPIC",
	}
end
```

Add the same block to the `bimg` project (around line 1430):

```lua
if (_OPTIONS["PLATFORM"]=="riscv64") then
	buildoptions {
		"-Wno-cast-align",
		"-mcmodel=medany",
		"-fPIC",
	}
end
```

Add the same block to the `bgfx` project (around line 1599):

```lua
if (_OPTIONS["PLATFORM"]=="riscv64") then
	buildoptions {
		"-Wno-cast-align",
		"-mcmodel=medany",
		"-fPIC",
	}
end
```

## Build Instructions

### 1. Set Environment Variables

```bash
export PKG_CONFIG_PATH=/path/to/sysroot/usr/lib/pkgconfig:/path/to/sysroot/usr/lib64/pkgconfig
```

Replace `/path/to/sysroot` with your actual sysroot path.

### 2. Build MAME

#### Recommended Build Command
With a properly configured sysroot including X11 and libXinerama:

```bash
make -j16 \
     CROSS_BUILD=1 \
     PLATFORM=riscv64 \
     ARCHITECTURE= \
     OVERRIDE_CC=riscv64-unknown-linux-gnu-gcc \
     OVERRIDE_CXX=riscv64-unknown-linux-gnu-g++ \
     PYTHON_EXECUTABLE=python3 \
     NOASM=1 \
     NO_USE_PORTAUDIO=1 \
     NO_USE_PIPEWIRE=1 \
     USE_QTDEBUG=0 \
     ARCHOPTS="--sysroot=/path/to/sysroot" \
     LDOPTS="--sysroot=/path/to/sysroot" \
     REGENIE=1
```

This enables all available features:
- ✅ OpenGL rendering support (via GLX)
- ✅ X11 window management
- ✅ SDL2 window management (fallback)
- ✅ PulseAudio audio output
- ✅ MIDI support via ALSA
- ✅ XInput for enhanced input devices

Only the following features are disabled (not commonly available in RISC-V sysroots):
- ❌ PortAudio (use SDL or PulseAudio instead)
- ❌ PipeWire (use SDL or PulseAudio instead)
- ❌ Qt debugger interface

**Note:** Replace `/path/to/sysroot` with your actual sysroot path (e.g., `/scratch/work/mame/sysroot`).

#### Minimal Build (SDL2 only, no X11)
If you want to build with minimal dependencies (SDL2 only, no X11/OpenGL/audio backends):

```bash
make -j16 \
     CROSS_BUILD=1 \
     PLATFORM=riscv64 \
     ARCHITECTURE= \
     OVERRIDE_CC=riscv64-unknown-linux-gnu-gcc \
     OVERRIDE_CXX=riscv64-unknown-linux-gnu-g++ \
     PYTHON_EXECUTABLE=python3 \
     NOASM=1 \
     NO_OPENGL=1 \
     NO_USE_PORTAUDIO=1 \
     NO_USE_PULSEAUDIO=1 \
     NO_USE_PIPEWIRE=1 \
     NO_USE_MIDI=1 \
     NO_USE_XINPUT=1 \
     USE_QTDEBUG=0 \
     NO_X11=1 \
     ARCHOPTS="--sysroot=/path/to/sysroot" \
     LDOPTS="--sysroot=/path/to/sysroot" \
     REGENIE=1
```

This uses SDL2 for video/audio only and disables all optional features.

**Note:** MAME can also run in headless mode without any graphics using `-video none`. This is useful for:
- Benchmarking emulation performance
- Running automated tests
- Command-line operations (e.g., `-listxml`, `-verifyroms`)
- Recording/playback of input files

Example headless usage:
```bash
./mame -video none -sound none -seconds_to_run 60 <system>  # Run for 60 seconds without video/audio
./mame -listxml > mame.xml                                   # Generate XML list (no graphics needed)
./mame -verifyroms pacman                                    # Verify ROM files (no graphics needed)
```

### 3. Verify Enabled Features

After running the build with `REGENIE=1`, you can check which features were detected:

```bash
# Check the generated build configuration
grep -E "USE_OPENGL|USE_MIDI|PULSEAUDIO|NO_X11|USE_XINPUT" build/projects/sdl/mame/gmake-linux/*.make
```

### 4. Build Output

The compiled MAME binary will be located at:
```
./mame
```

## Build Options Explained

### Required Options
- **CROSS_BUILD=1**: Enable cross-compilation mode
- **PLATFORM=riscv64**: Target RISC-V 64-bit architecture
- **ARCHITECTURE=**: Leave empty to avoid x86-specific flags
- **OVERRIDE_CC/OVERRIDE_CXX**: Specify RISC-V cross-compiler
- **NOASM=1**: Disable assembly optimizations (not available for RISC-V)
- **ARCHOPTS**: Compiler options (sysroot path)
- **LDOPTS**: Linker options (sysroot path)
- **REGENIE=1**: Regenerate build files (use on first build or after changes)

### Feature Control Options

#### Always Disabled (libraries not commonly available in RISC-V sysroots)
- **NO_USE_PORTAUDIO=1**: Disable PortAudio audio backend
- **NO_USE_PIPEWIRE=1**: Disable PipeWire audio backend
- **USE_QTDEBUG=0**: Disable Qt debugger interface

#### Auto-Detected (enabled if libraries are in sysroot)
The following features are automatically enabled if the corresponding libraries are found in your sysroot:

- **OpenGL Support**: Enabled if `libGL.so` and GL headers are present
  - Provides hardware-accelerated rendering
  - Disable with `NO_OPENGL=1` if not available

- **X11 Window Management**: Enabled if `libX11.so`, `libXinerama.so` and X11 headers are present
  - Provides native X11 window support in addition to SDL
  - **Important**: Requires BOTH libX11 AND libXinerama (see installation instructions above)
  - Disable with `NO_X11=1` only if you want SDL-only mode

- **PulseAudio Support**: Enabled if `libpulse.so` and PulseAudio headers are present
  - Provides PulseAudio audio output
  - Disable with `NO_USE_PULSEAUDIO=1` if not available

- **MIDI Support**: Enabled if `libasound.so` (ALSA) and ALSA headers are present
  - Provides MIDI device support via ALSA
  - Disable with `NO_USE_MIDI=1` if not available

- **XInput Support**: Enabled if XInput headers and libraries are present
  - Provides enhanced input device support (tablets, multi-touch, etc.)
  - Disable with `NO_USE_XINPUT=1` if not available

## Troubleshooting

### Clean Build
If you encounter issues, try a clean rebuild:
```bash
make clean
# Then run the build command again with REGENIE=1
```

### Relocation Errors
If you see errors like `relocation truncated to fit: R_RISCV_PCREL_HI20`:
- Ensure `-mcmodel=medany` and `-fPIC` flags are added to all RISC-V sections in the patch files
- Clean and rebuild the affected libraries:
  ```bash
  rm -rf build/projects/sdl/mame/gmake-linux/obj/Release/3rdparty/bx/
  rm -rf build/projects/sdl/mame/gmake-linux/obj/Release/3rdparty/bimg/
  rm -rf build/projects/sdl/mame/gmake-linux/obj/Release/3rdparty/bgfx/
  ```

### SDL2_ttf Version Mismatch
If you get undefined reference errors for SDL functions with `@SUSE_*` symbols:
- Ensure your SDL2 and SDL2_ttf libraries are from the same distribution/version
- Both runtime and development packages must be installed in the sysroot

### Missing Libraries
If the linker cannot find libraries:
- Verify `PKG_CONFIG_PATH` is set correctly
- Check that libraries exist in sysroot: `find /path/to/sysroot -name "lib*.so"`
- Ensure both `usr/lib` and `usr/lib64` are in the sysroot library paths

### Missing libXinerama Error
If you get an error like `cannot find -lXinerama: No such file or directory`:
- This means libXinerama is not installed in your sysroot
- **Solution**: Install libXinerama using the instructions in the "Installing Libraries to Sysroot" section above
- Verify installation: `ls /path/to/sysroot/usr/lib64/libXinerama.so`
- Note: You may have `libxcb-xinerama.so` (XCB version) but MAME needs `libXinerama.so` (Xlib version)

## Technical Details

### RISC-V Specific Modifications

1. **Alignment Warnings**: RISC-V has stricter alignment requirements than x86. The `-Wno-cast-align` flag suppresses warnings for pointer casts that may increase alignment requirements.

2. **Code Model**: The `-mcmodel=medany` flag allows code to be placed anywhere in the address space within a 2GB range, which is necessary for larger binaries like MAME.

3. **Position Independent Code**: The `-fPIC` flag generates position-independent code, which helps avoid relocation issues during linking.

4. **Feature Availability**: The following features depend on your sysroot contents:
   - **Always Available**: SDL2 video/audio, software rendering, bgfx rendering
   - **Conditionally Available** (if libraries in sysroot):
     - OpenGL hardware-accelerated rendering
     - X11 window management
     - PulseAudio audio output
     - MIDI support via ALSA
     - XInput enhanced input devices
   - **Not Available**: PortAudio, PipeWire, Qt debugger (rarely in RISC-V sysroots)

5. **Video Backend Options**: MAME supports multiple video backends that can be selected at runtime:
   - **`-video soft`**: Software rendering (default on Linux, works everywhere)
   - **`-video opengl`**: OpenGL hardware acceleration (requires OpenGL libraries)
   - **`-video bgfx`**: Modern cross-platform renderer with shader support
   - **`-video none`**: No video output (for benchmarking, testing, or command-line operations)

   X11 is only used for window management when available. MAME can use SDL2 for windowing even without X11 (e.g., on Wayland or framebuffer).

### File Modifications Summary

| File | Purpose | Changes |
|------|---------|---------|
| `scripts/genie.lua` | Main MAME build configuration | Add RISC-V compiler flags |
| `scripts/src/3rdparty.lua` | Third-party library configuration | Add RISC-V flags to bx, bimg, bgfx |

## Testing

After building, you can verify the binary:
```bash
file ./mame
# Should output: mame: ELF 64-bit LSB executable, UCB RISC-V, ...

# Check dynamic library dependencies
riscv64-unknown-linux-gnu-readelf -d ./mame | grep NEEDED
```

## Running MAME on RISC-V

Transfer the `mame` binary to your RISC-V system along with:
- Required shared libraries from your sysroot (SDL2, SDL2_ttf, X11, etc.)
- MAME ROM files
- MAME data files (hash, artwork, etc.)

Example:
```bash
./mame -listxml  # List all supported systems
./mame <system>  # Run a specific system
```

## Checking Your Sysroot Capabilities

To determine which features are available in your sysroot, run:

```bash
# Check for OpenGL support
find /path/to/sysroot -name "libGL.so*" -o -name "gl.h"

# Check for X11 support
find /path/to/sysroot -name "libX11.so*" -o -name "x11.pc"

# Check for PulseAudio support
find /path/to/sysroot -name "libpulse.so*" -o -name "libpulse.pc"

# Check for ALSA/MIDI support
find /path/to/sysroot -name "libasound.so*" -o -name "alsa.pc"

# Check for XInput support
find /path/to/sysroot -name "XInput.h" -o -name "libxcb-xinput.so*"

# List all available pkg-config files
ls /path/to/sysroot/usr/lib/pkgconfig/ /path/to/sysroot/usr/lib64/pkgconfig/
```

Based on the results, you can decide which `NO_*` flags to include in your build command.

## Known Limitations

- PortAudio and PipeWire are typically not available in RISC-V sysroots
- Qt debugger interface requires Qt5 development libraries (rarely available)
- Performance may vary depending on RISC-V hardware capabilities
- Some features require specific libraries in the sysroot (see "Checking Your Sysroot Capabilities" above)

## Contributing

If you improve the RISC-V build process or add support for additional features, please consider contributing back to the MAME project.

## License

MAME is distributed under the terms of the GNU General Public License, version 2 or later (GPL-2.0+). See the main MAME documentation for full license details.


