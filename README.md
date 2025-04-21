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

*This will be a wild ride since I've never used CI before. lmk if there are better practise.*

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
Among them, brotli can compile w/o libjxl or highway (patched).

I'm try if newer tools can help, so Boost is now 1.87 instead of 1.80. To revert simply change the numbers and enable the patches.

First, build Qt and its dependencies:
meson, pkg_config, zlib, boost, png, jpeg, icu, and googleangle

Then, follow latest/krita-deps.yml to install.
Before compile libintl, compile and install gettext.

*Note: freetype compiles but seems to have issue. I've disabled in qt5 but Krita itself still has this dependency. You may still need to re-enabled in qt5*

## Compile Krita ##

Make sure you have the _install from previous step in an appropriate place and use it.

This will compile and produce Krita exe. I have not tested with Clang19 but Clang18 works. 

**Icoutils does not compile, so I copied the Krita installation's icon and comtinued the build.**

## Testing ##

w/ Clang18 and system freetype in qt5

         24 - libs-flake-TestPathTool (Failed)
         47 - libs-flake-TestSvgParser (Failed)
         48 - libs-flake-TestSvgParserCloned (Failed)
         49 - libs-flake-TestSvgParserRoundTrip (Failed)
         51 - libs-pigment-TestKoColorSet (ILLEGAL)
        166 - libs-image-kis_walkers_test (Failed)
        167 - libs-image-kis_cage_transform_worker_test (Failed)
        229 - libs-ui-kis_shape_controller_test (SEGFAULT)
        230 - libs-ui-kis_dummies_facade_test (SEGFAULT)
        265 - plugins-dockers-animation-timeline_model_test (Failed)
        271 - plugins-generators-kis_seexpr_generator_test (Failed)
        273 - plugins-impex-kis_kra_saver_test (SEGFAULT)
        275 - plugins-impex-kis_tiff_test (Failed)
        291 - plugins-impex-KisHeifTest (Failed)

qt5-base has many failed tests by itself, even with x64 build. On Windows on Arm, some tests fail, but not with the debug test exe. Not sure why.