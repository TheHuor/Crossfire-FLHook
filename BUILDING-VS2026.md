# Build with Visual Studio 2026 (Windows x86)

This guide uses the VS2026 v145 compiler. For the requested VS2022 v143 compiler
inside the VS2026 IDE, use [BUILDING-V143.md](BUILDING-V143.md).

FLHook runs inside the 32-bit Freelancer server. Use the x86 compiler even on
a 64-bit Windows installation. Install Visual Studio's Desktop development
with C++ workload, including the Windows SDK and CMake tools, and Python 3.12+.

Run the following from this repository in PowerShell:

```powershell
git submodule update --init --recursive
python -m venv .venv
& ./.venv/Scripts/Activate.ps1
python -m pip install conan==2.33.0 click==8.5.0

$vswhere = "${env:ProgramFiles(x86)}/Microsoft Visual Studio/Installer/vswhere.exe"
$vs = & $vswhere -latest -products * -requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64 -property installationPath
Import-Module "$vs/Common7/Tools/Microsoft.VisualStudio.DevShell.dll"
Enter-VsDevShell -VsInstallPath $vs -SkipAutomaticLocation -DevCmdArguments '-arch=x86 -host_arch=x64'
$env:PATH = "$vs/Common7/IDE/CommonExtensions/Microsoft/CMake/CMake/bin;$vs/Common7/IDE/CommonExtensions/Microsoft/CMake/Ninja;$env:PATH"
$env:CONAN_HOME = "$PWD/.conan2"
conan profile detect
conan install . -pr:b=default -pr:h=./profiles/windows-vs2026 -s build_type=Release --build=missing
if ($LASTEXITCODE) { throw 'Conan dependency build failed' }

cmake -S . -B build/Release -G Ninja '-DCMAKE_TOOLCHAIN_FILE=build/Release/generators/conan_toolchain.cmake' '-DCMAKE_BUILD_TYPE=Release' '-DCOMPILE_PLUGINS=ON' '-DCOMPILE_EXAMPLES=ON' '-DNO_POST_BUILD=TRUE'
if ($LASTEXITCODE) { throw 'CMake configuration failed' }
cmake --build build/Release --parallel 4
if ($LASTEXITCODE) { throw 'FLHook build failed' }
python cli.py post_build --release
```

The main DLL and dependency DLLs are in `build/Release/bin`; plugin DLLs are in
`build/Release/bin/plugins`. The final command collects the DLLs into `dist`
and creates `build.zip`. The first dependency build can take a while, especially
for ICU and OpenSSL; subsequent builds reuse the repository-local Conan cache.

## Editing and building in Visual Studio 2026

The repository-root `FLHook.sln` contains only the C# SaveConverter utility.
After installing the Conan dependencies above, generate the C++ solution with:

```powershell
cmake -S . -B build/VS2026 -G 'Visual Studio 18 2026' -A Win32 '-DCMAKE_TOOLCHAIN_FILE=../Release/generators/conan_toolchain.cmake' '-DCMAKE_CONFIGURATION_TYPES=Release' '-DCOMPILE_PLUGINS=ON' '-DCOMPILE_EXAMPLES=ON' '-DNO_POST_BUILD=TRUE'
if ($LASTEXITCODE) { throw 'Visual Studio solution generation failed' }
cmake --build build/VS2026 --config Release --parallel 4
if ($LASTEXITCODE) { throw 'Visual Studio build failed' }
Copy-Item build/Release/bin/*.dll build/VS2026/bin/ -Exclude FLHook.dll
```

Open `build/VS2026/FLHook.slnx` in Visual Studio 2026 and select **Release | Win32**.
Use **Build Solution** (Ctrl+Shift+B) to build FLHook and every enabled plugin.
Edit the source files through the solution as usual. Make changes to build
settings in `CMakeLists.txt` rather than generated `.vcxproj` files; CMake can
regenerate those files. The native solution writes DLLs to `build/VS2026/bin`
and `build/VS2026/bin/plugins`. Only Release is configured because the installed
Conan dependencies are Release binaries.

This uses the existing locked dependency versions and adds a separate MSVC 195
profile for Visual Studio 2026. The original `profiles/windows` remains available
for MSVC 194 / Visual Studio 2022. A successful build does not verify integration
with a running Freelancer server.
