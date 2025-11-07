# Building AtomicLinkRPC from Source

This guide provides detailed instructions for building AtomicLinkRPC (ALR) from source on Windows, Linux, and macOS.

## Prerequisites

Before building ALR, ensure your development environment meets the following requirements:

### Required Dependencies

- **C++20 Compiler**: A modern C++ compiler with C++20 support
    - GCC 10+ on Linux
    - Clang 10+ on macOS/Linux
    - MSVC 2019+ or Clang on Windows
- **CMake**: Version 3.15 or higher
- **Ninja** (recommended): Fast build system
- **LLVM/Clang with libclang**: Required for the ALR code generator

### Optional Dependencies

- **OpenSSL**: Required for TLS/SSL support (enabled by default)
  - Can be disabled by setting `-DUSE_OPENSSL=OFF` during CMake configuration

## Installing Dependencies

### Windows

#### Option 1: Using vcpkg (Recommended)

```powershell
# Install vcpkg
git clone https://github.com/Microsoft/vcpkg.git
cd vcpkg
.\bootstrap-vcpkg.bat

# Install dependencies
.\vcpkg install openssl:x64-windows-static
.\vcpkg install llvm:x64-windows

# Integrate with Visual Studio (optional)
.\vcpkg integrate install
```

#### Option 2: Using Pre-built Binaries

1. **CMake**: Download from [cmake.org](https://cmake.org/download/)
2. **Ninja**: Download from [ninja-build.org](https://ninja-build.org/) or install via chocolatey:
   ```powershell
   choco install ninja
   ```
3. **LLVM/Clang**: Download from [llvm.org](https://llvm.org/builds/)
   - **Important**: Ensure the distribution includes `libclang` (typically in `bin/libclang.dll`)
4. **OpenSSL**: Download from [slproweb.com](https://slproweb.com/products/Win32OpenSSL.html)

#### Setting Environment Variables

Set the `CLANG_PATH` environment variable to point to your Clang installation:

```powershell
# PowerShell
$env:CLANG_PATH = "C:\Program Files\LLVM"
# Or permanently via System Properties > Environment Variables
```

### Linux (Ubuntu/Debian)

```bash
# Update package lists
sudo apt update

# Install build tools
sudo apt install -y build-essential cmake ninja-build git

# Install Clang/LLVM with libclang
sudo apt install -y llvm-14 clang-14 libclang-14-dev

# Install OpenSSL
sudo apt install -y libssl-dev

# Set CLANG_PATH (add to ~/.bashrc for persistence)
export CLANG_PATH=/usr/lib/llvm-14
```

### Linux (Fedora/RHEL)

```bash
# Install build tools
sudo dnf install -y gcc-c++ cmake ninja-build git

# Install Clang/LLVM with libclang
sudo dnf install -y llvm clang clang-devel

# Install OpenSSL
sudo dnf install -y openssl-devel

# Set CLANG_PATH
export CLANG_PATH=/usr
```

### macOS

```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install dependencies
brew install cmake ninja llvm openssl

# Set CLANG_PATH (add to ~/.zshrc or ~/.bash_profile for persistence)
export CLANG_PATH=$(brew --prefix llvm)

# You may also need to set OpenSSL paths
export OPENSSL_ROOT_DIR=$(brew --prefix openssl)
```

## Cloning the Repository

Clone the AtomicLinkRPC repository from GitHub:

```bash
git clone https://github.com/alr-project/AtomicLinkRPC.git
cd AtomicLinkRPC/repo
```

## Building ALR

### Windows Build

Using Clang (recommended for best performance):

```powershell
# Navigate to repository root
cd AtomicLinkRPC\repo

# Configure the build
cmake -S . -B build `
  -DCMAKE_BUILD_TYPE=Release `
  -G "Ninja" `
  -DCMAKE_C_COMPILER=clang `
  -DCMAKE_CXX_COMPILER=clang++ `
  -DCMAKE_INSTALL_PREFIX=build

# Build the project (adjust -j value based on your CPU cores)
cmake --build build --config Release -j 8

# Install binaries to build/bin
cmake --install build --config Release
```

Using MSVC:

```powershell
cmake -S . -B build `
  -DCMAKE_BUILD_TYPE=Release `
  -G "Visual Studio 17 2022" `
  -A x64 `
  -DCMAKE_INSTALL_PREFIX=build

cmake --build build --config Release -j 8
cmake --install build --config Release
```

### Linux Build

```bash
# Navigate to repository root
cd AtomicLinkRPC/repo

# Configure the build
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -G "Ninja" \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DCMAKE_INSTALL_PREFIX=build

# Build the project
cmake --build build --config Release -j $(nproc)

# Install binaries
cmake --install build --config Release
```

### macOS Build

```bash
# Navigate to repository root
cd AtomicLinkRPC/repo

# Configure the build
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -G "Ninja" \
  -DCMAKE_C_COMPILER=$(brew --prefix llvm)/bin/clang \
  -DCMAKE_CXX_COMPILER=$(brew --prefix llvm)/bin/clang++ \
  -DCMAKE_INSTALL_PREFIX=build

# Build the project
cmake --build build --config Release -j $(sysctl -n hw.ncpu)

# Install binaries
cmake --install build --config Release
```

## Build Output

After a successful build, you'll find:

- **Executables**: `build/` directory
    - `alr_codegen/alr_codegen` (or `alr_codegen/alr_codegen.exe` on Windows) - The ALR code generator
    - Example applications (e.g., `bin/city_guide_client`, `bin/city_guide_service`)
- **Generated Code**: `build/<project_name>/gen/cpp/` directories
- **Build Artifacts**: `build/` directory

## Selective Builds

### Building Specific Projects

You can selectively build specific components by editing `repo/CMakeLists.txt` and commenting out unwanted projects:

```cmake
# filepath: repo/CMakeLists.txt
include(${CMAKE_CURRENT_LIST_DIR}/cmake/ALRCore.cmake)

# Comment out projects you don't want to build
add_subdirectory(tests/cpp/perf1 "cmake/perf1")
# add_subdirectory(tests/cpp/perf2 "cmake/perf2")
add_subdirectory(examples/cpp/city_guide "cmake/city_guide")
```

### Building Specific Targets

Alternatively, use the `--target` option to build only specific executables:

```bash
# Build only the CityGuide client and service
cmake --build build --config Release --target city_guide_client city_guide_service

# Build only the code generator
cmake --build build --config Release --target alr_codegen
```

To see all available targets:

```bash
cmake --build build --target help
```

## Build Configuration Options

### Disabling OpenSSL/TLS Support

If you don't need TLS support:

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DUSE_OPENSSL=OFF \
  -G "Ninja"
```

### Debug Build

For development and debugging:

```bash
cmake -S . -B build_debug \
  -DCMAKE_BUILD_TYPE=Debug \
  -G "Ninja"

cmake --build build_debug
```

### Verbose Build Output

To see detailed compilation commands:

```bash
cmake --build build --config Release --verbose
```

Or set the CMake variable:

```bash
cmake -S . -B build -DCMAKE_VERBOSE_MAKEFILE=ON
```

## Verifying the Build

After building, verify that the code generator works:

```bash
# On Windows
.\build\alr_codegen\alr_codegen.exe --version

# On Linux/macOS
./build/alr_codegen/alr_codegen --version
```

Run the CityGuide example:

```bash
# Start the service (in one terminal)
./build/bin/city_guide_service

# Run the client (in another terminal)
./build/bin/city_guide_client
```

## Troubleshooting

### libclang Not Found

**Symptom**: `alr_codegen` fails with "libclang not found" error.

**Solution**: Ensure `CLANG_PATH` is set correctly and points to a directory containing `libclang`:

```bash
# Verify the library exists
ls $CLANG_PATH/bin/libclang*    # Linux/macOS
dir %CLANG_PATH%\bin\libclang*  # Windows
```

### OpenSSL Linking Issues

**Symptom**: Build fails with OpenSSL-related errors.

**Solution**: 
- On Windows with vcpkg: Ensure vcpkg is properly integrated
- On Linux: Install `libssl-dev` or `openssl-devel`
- On macOS: Set `OPENSSL_ROOT_DIR` to Homebrew's OpenSSL location

### CMake Version Too Old

**Symptom**: CMake reports version incompatibility.

**Solution**: Update CMake to version 3.15 or higher:

```bash
# Linux
sudo apt install cmake  # or download from cmake.org

# macOS
brew upgrade cmake

# Windows
# Download installer from cmake.org
```

### Clang/LLVM Missing libclang

**Symptom**: ALR codegen cannot find libclang even with `CLANG_PATH` set.

**Solution**: Some LLVM distributions don't include libclang. You may need to:
1. Build LLVM from source with libclang enabled
2. Install a development package (e.g., `libclang-dev` on Ubuntu)
3. Use a different LLVM distribution that includes libclang

## Next Steps

- Read the [User Guide](user-guide.md) to learn how to use ALR
- Explore the [Examples](examples.md) to see ALR in action
- Check the [API Reference](api-reference-core.md) for detailed API documentation
- Review [Technical Documentation](technical.md) for architecture details
