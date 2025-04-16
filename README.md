# Krita-WoA-Port
Porting Krita to Windows on Arm64

*Native build for now


## Prepareation

1. Install Visual Studio 2022 v17.13.5
2. Compile and Install Python 3.10
3. Install StrawberryPerl (I couldn't find an Arm64 alternative, maybe fine?)
4. Compile OpenSSL1.1.1w w/ patch using MSVC
5. git, cmake, ninja, llvm as [link](https://docs.krita.org/en/untranslatable_pages/building_krita.html#prerequisites) says

    *You may want to use newer cmake to avoid putting cmake_policy(SET CMP0074 NEW) everywhere (I have omitted them)*

6. Download and install ccache and patch
6. Keep following the website and activate PythonEnv
7. Install lxml-4.9.3-py3.10-win-arm64.egg in PyhonEnv
8. Keep following the website and install the other python requirements

## Krita-deps-management

Command to run all builds *Not really working somehow*: 
**python -u ci-utilities/seed-package-registry.py --seed-file latest/krita-deps.yml --platform Windows/Qt5/Shared --skip-dependencies-fetch**

Command to compile one dependency: **python -u ..\ci-utilities\run-ci-build.py --project ext_\<name> --branch master --platform Windows/Qt5/Shared --only-build --fail-on-leaked-stage-files --skip-dependencies-fetch**

Since we are not using the default pre-compiled dependencies, we have to manually create directories.

Create _install, cache, and downloads

Link _install to subdirectories

Move OpenSSL binaries/libraries into _install accordingly

copy repo-metadata to ci-ultilities

set environment vars: LLVM_INSTALL_DIR KDECI_CACHE_PATH(to cache directory created) EXTERNALS_DOWNLOAD_DIR(to download directory created)

The dependencies has dependencies among them, however the latest/krita.dep does not have them in a nice order. You will have to go back and forth between compile all and compile one to resolve them. (Currently not really usable anyway. I may have missed many env vars)

There are a few dependencies that won't compile for Windows on Arm, commented out in latest/krita.dep, check diff to find out.
Among them, freetype compiles but seems to have issue (removed from qt, you may want to experiment with it), and brotli can compile w/o libjxl or highway (patched).

I'm try if newer tools can help, so Boost is now 1.87 instead of 1.80. To revert simply change the numbers and enable the patches.

Currently only cleaned up upon building Qt.
You will need: meson, pkg_config, zlib, boost, png, jpeg, icu, and googleangle



