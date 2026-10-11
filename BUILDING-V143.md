# Visual Studio 2026 IDE with the Visual Studio 2022 compiler (v143)

Use the VS2026 IDE with the optional **MSVC v143 - VS 2022 C++ x64/x86 build
tools (v14.44-17.14)** component
(`Microsoft.VisualStudio.Component.VC.14.44.17.14.x86.x64`). Select this versioned
component: the similarly named component group alone does not install the native
v143 MSBuild integration. FLHook must be built for **Win32**, because Freelancer
is a 32-bit application. The Windows SDK and CMake tools must also be installed.

The repository-root `FLHook.sln` contains only the C# SaveConverter utility.
The C++ solution generated below is `build/VS2026-v143/FLHook.slnx`.

## Prepare dependencies

Run from the repository root in PowerShell with Python 3.12+:

```powershell
git submodule update --init --recursive
python -m venv .venv
& ./.venv/Scripts/Activate.ps1
python -m pip install conan==2.33.0 click==8.5.0

$vswhere = "${env:ProgramFiles(x86)}/Microsoft Visual Studio/Installer/vswhere.exe"
$vs = & $vswhere -latest -products * -requires Microsoft.VisualStudio.Component.VC.14.44.17.14.x86.x64 -property installationPath
if (!$vs) { throw 'Install the optional v143 x86/x64 build tools in VS2026 first' }
$env:PATH = "$vs/Common7/IDE/CommonExtensions/Microsoft/CMake/CMake/bin;$vs/Common7/IDE/CommonExtensions/Microsoft/CMake/Ninja;$env:PATH"
$env:CONAN_HOME = "$PWD/.conan2"
conan profile detect --force
conan install . -of build/v143 -pr:b=default -pr:h=./profiles/windows-vs2026-v143 -s build_type=Release --build=missing --lockfile-partial --lockfile-out=build/v143/conan.lock
if ($LASTEXITCODE) { throw 'Conan dependency build failed' }
```

The host profile fixes the compiler to MSVC 19.44 / v143 while selecting the
VS2026 installation. JOM builds OpenSSL in parallel. `--lockfile-partial` permits
this additional build tool; existing library revisions remain locked by the
repository's `conan.lock`. Subsequent dependency installs can use
`--lockfile=build/v143/conan.lock` to reuse the full generated lockfile.

## Generate and build the solution

Continue in the same PowerShell session:

```powershell
cmake -S . -B build/VS2026-v143 -G 'Visual Studio 18 2026' -A Win32 -T v143 '-DCMAKE_TOOLCHAIN_FILE=../v143/build/Release/generators/conan_toolchain.cmake' '-DCMAKE_CONFIGURATION_TYPES=Release' '-DCOMPILE_PLUGINS=ON' '-DCOMPILE_EXAMPLES=ON' '-DNO_POST_BUILD=TRUE'
if ($LASTEXITCODE) { throw 'Solution generation failed' }
cmake --build build/VS2026-v143 --config Release --parallel 4
if ($LASTEXITCODE) { throw 'FLHook build failed' }
Copy-Item build/v143/build/Release/bin/*.dll build/VS2026-v143/bin/ -Exclude FLHook.dll
```

Open **build/VS2026-v143/FLHook.slnx**, select **Release | Win32**, and use
**Build Solution** (Ctrl+Shift+B). FLHook and all enabled plugins are included.
The project properties must show **Platform Toolset: v143**.

The solution uses the folders **FLHook**, **Plugins**, **Plugins/Examples**,
**Dependencies**, and **CMake**. Their assignments are defined in CMake and survive
regeneration of the solution.

Edit source files through the solution as usual. Change build settings in the
repository's `CMakeLists.txt`, since CMake can regenerate `.vcxproj` files.
Only Release is configured; Debug needs separate Debug dependencies.

## Output

- Main DLL and dependency DLLs: `build/VS2026-v143/bin`
- Plugin DLLs: `build/VS2026-v143/bin/plugins`
- Conan executable: `.venv/Scripts/conan.exe`
- Dependency cache: `.conan2`
- v143 dependency DLLs: `build/v143/build/Release/bin`

Keep v143 and v145 artifacts separate. The general `cli.py post_build` command
searches all build directories; when both toolsets have been built, collect the
DLLs specifically from `build/VS2026-v143/bin` for a v143 deployment.

To package the v143 build from this checkout:

```powershell
New-Item -ItemType Directory -Force dist/plugins | Out-Null
Get-ChildItem build/VS2026-v143/bin -Filter '*.dll' | Copy-Item -Destination dist -Force
Get-ChildItem build/VS2026-v143/bin/plugins -Filter '*.dll' | Copy-Item -Destination dist/plugins -Force
Compress-Archive -Path dist/* -DestinationPath build.zip -Force
```

Successful compilation does not verify runtime behavior inside FLServer.
