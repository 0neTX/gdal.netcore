## Windows build scripts for GDAL

## Table of contents

- [Windows build scripts for GDAL](#windows-build-scripts-for-gdal)
  * [Prerequisites:](#prerequisites)
  * [Building: (in PowerShell)](#building-in-powershell)
  * [Platform notes:](#platform-notes)
  * [Troubleshooting dependencies:](#troubleshooting-dependencies)

Table of contents generated with [markdown-toc](http://ecotrust-canada.github.io/markdown-toc/).

In this folder contains Powershell and NMake scripts for building Windows runtime packages.

### Prerequisites:

1. [Visual Studio Build Tools (with ATL)](https://visualstudio.microsoft.com/thank-you-downloading-visual-studio/?sku=BuildTools&rel=17). **vs17.0(2022) or greater is required** (**nmake** and to retarget **libpng** to v143 toolset). The build initializes the 2022 environment explicitly rather than letting the toolset be auto-detected: the `arrow` dependency pulls in `xsimd`, whose C++20 templates do not compile under the vs16.0(2019) toolset, so a machine with both installed must not fall back to 2019.

2. [.NET Core SDK](https://dotnet.microsoft.com/en-us/download/dotnet/7.0) and [Nuget.exe](https://docs.microsoft.com/en-us/nuget/install-nuget-client-tools) - for building and publishing packages respectively.

The Windows build uses the same shared VCPKG manifest authority as Unix and macOS:

- `../shared/vcpkg.json`
- `../shared/vcpkg-configuration.json`
- `../shared/vcpkg-lock.json`

### Building: (in PowerShell)

1. Call `./install.ps1` to install all required packages and tools. <br/>
Possible options:
   ```powershell
    [bool] $cleanGdalBuild = $true,  # clean gdal-build folder (output) before build
    [bool] $cleanGdalIntermediate = $true, # clean gdal-cmake-temp (cache) folder
    [bool] $cleanProjBuild = $true, # clean proj-build folder (output) before build    
    [bool] $cleanProjIntermediate = $true, # clean proj-cmake-temp (cache) folder
    [bool] $bootstrapVcpkg = $true, # bootstrap VCPKG from scratch
    [bool] $installVcpkgPackages = $true, # install VCPKG packages
    [bool] $isDebug = $false # build debug version of packages
    ```
   Example: 
   ```powershell 
   ./install.ps1 -cleanGdalBuild:$true -cleanGdalIntermediate:$true -isDebug:$true
   ```
This will install all required VCPKG packages and tools, build GDAL and PROJ, and build runtime and core packages.

On CI, cache-warm runs restore the VCPKG archive cache separately from the Windows build-output cache. When the manifest/build inputs match, the build-state stamps allow the workflow to reuse cached VCPKG, PROJ, and GDAL outputs instead of recompiling them from scratch.

2. Call `./test.ps1` to test runtime and core packages. <br/> 
If everything runs smoothly, you can use a local nuget feed to include packages in your project.

### Platform notes:

- **PostgreSQL/PostGIS drivers are not built on Windows.** GDAL is configured with
  `-DGDAL_USE_POSTGRESQL=OFF` here, so the `PostgreSQL` and `PostGISRaster` drivers are
  absent from the Windows runtime package (they remain available on Linux and macOS).
  The reason is provenance: `../shared/vcpkg.json` has no `libpq` in its `windows-dynamic`
  feature, so CMake resolved PostgreSQL against the frozen GisInternals SDK and the
  package ended up shipping that SDK's outdated `LIBPQ.dll`. The `PGDump` (SQL dump) and
  `PGeo` (ODBC) drivers do not use libpq and are still available.
  See [issue #241](https://github.com/MaxRev-Dev/gdal.netcore/issues/241).

### Troubleshooting dependencies:
Use **dumpbin** or [**dependency walker**](https://www.dependencywalker.com/) to check gdal's dependencies. Please ensure the tests are passing before bringing them to prod.

#### SWIG reports `Unable to find 'swig.swg'`

SWIG locates its `.swg` library relative to the executable it was invoked through, so an
installation reached via a package manager shim breaks it. WinGet, for example, puts a
symlink at `%LOCALAPPDATA%\Microsoft\WinGet\Links\swig.exe`; SWIG then looks for its
library under `Links\Lib`, which does not exist, and the C# binding generation fails while
CMake still reports SWIG as found.

The build resolves the symlink and passes the real binary to CMake via `SWIG_EXECUTABLE`.
To check what an installation resolves to:

```powershell
swig -swiglib   # must print a directory that actually contains swig.swg
```

Note that the environment variable SWIG reads is `SWIG_LIB`, not `SWIGLIB`, and that SWIG
has no `-L` option for this — library directories are added with `-I`.

Have fun)

Contact me: [Telegram - MaxRev](http://t.me/maxrev)
