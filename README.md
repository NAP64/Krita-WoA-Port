# Krita-WoA-Port
Porting Krita to Windows on Arm64

*Native build for now


## Prepareation

1. Install Visual Studio 2022 v17.13.5
2. Compile and Install Python 3.10

    python.bat PC/layout --precompile --preset-default --copy \<install path\>

3. Install StrawberryPerl (I couldn't find an Arm64 alternative, maybe fine?)
4. Patch and compile OpenSSL1.1.1w w/ patch using MSVC

    install to the _install dir of choice, change lib\\\*.lib to lib\\\*.dll.a

5. git, cmake, ninja, llvm as [link](https://docs.krita.org/en/untranslatable_pages/building_krita.html#prerequisites) says

    *You may want to use newer cmake to avoid putting cmake_policy(SET CMP0074 NEW) everywhere (I have omitted them)*

6. Download and install ccache and patch
6. Keep following the website and activate PythonEnv
7. Install lxml-4.9.3-py3.10-win-arm64.egg in PyhonEnv
8. Keep following the website and install the other python requirements

## Krita-deps-management

*This will be a wild ride since I've never used CI before. lmk if there are better practise.*

Apply the patch: git apply krita-deps-management\000-krita-deps.patch

Run: python krita-deps-management\tools\setup-env.py *-d* -v PythonEnv -p \<llvm-mingw-path\>\bin\ -p \<llvm-mingw-path\>\aarch64-w64-mingw32\bin\ -p \<ninja-path\>

Run: base-env.bat

Set your _install dir of choice as KDECI_SHARED_INSTALL_PATH

Copy repo-metadata to ci-ultilities

Set environment vars for QT: LLVM_INSTALL_DIR

Run the command to run all builds: 
**python -u ci-utilities/seed-package-registry.py --seed-file latest/krita-deps.yml --platform Windows/Qt5/Shared --skip-dependencies-fetch**

Command to compile one dependency, for reference: **python -u ..\ci-utilities\run-ci-build.py --project ext_\<name> --branch master --platform Windows/Qt5/Shared --only-build --fail-on-leaked-stage-files --skip-dependencies-fetch**

There are a few dependencies that won't compile for Windows on Arm, commented out in latest/krita.dep, check diff to find out.
Among them, brotli can compile w/o libjxl or highway (patched).

I'm trying if newer tools can help, so Boost is now 1.87 instead of 1.80. To revert simply change the numbers and enable the patches.

*Note: freetype compiles but seems to have issue, but required. Leaving as-is.*

## Compile Krita ##

Point -DMAKE_INSTALL_PREFIX to your _install dir of choice, Krita will compile normally with one simple fix: Copy krita.ico and kritafile.ico from a Krita installation to b_krita/krita.
This is needed since icoutils won't compile for aarch64.

Might need to remove benchmarks/kis_composition_benchmark_NEON64.cpp:643 for Clang 18 and 19.

## Testing ##

w/ Clang18

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

w/ Clang19

         24 - libs-flake-TestPathTool (Failed)
         47 - libs-flake-TestSvgParser (Failed)
         48 - libs-flake-TestSvgParserCloned (Failed)
         49 - libs-flake-TestSvgParserRoundTrip (Failed)
         51 - libs-pigment-TestKoColorSet (ILLEGAL)
        166 - libs-image-kis_walkers_test (Failed)
        167 - libs-image-kis_cage_transform_worker_test (Failed)
        215 - libs-ui-kis_shape_layer_test (Failed)
        229 - libs-ui-kis_shape_controller_test (SEGFAULT)
        230 - libs-ui-kis_dummies_facade_test (SEGFAULT)
        265 - plugins-dockers-animation-timeline_model_test (Failed)
        271 - plugins-generators-kis_seexpr_generator_test (Failed)
        273 - plugins-impex-kis_kra_saver_test (SEGFAULT)

Looks like some errors are due to compiler.

w/ Clang20

         24 - libs-flake-TestPathTool (Failed)
         47 - libs-flake-TestSvgParser (Failed)
         48 - libs-flake-TestSvgParserCloned (Failed)
         49 - libs-flake-TestSvgParserRoundTrip (Failed)
         51 - libs-pigment-TestKoColorSet (ILLEGAL)
        166 - libs-image-kis_walkers_test (Failed)
        167 - libs-image-kis_cage_transform_worker_test (Failed)
        216 - libs-ui-KisSafeDocumentLoaderTest (Failed)
        229 - libs-ui-kis_shape_controller_test (SEGFAULT)
        230 - libs-ui-kis_dummies_facade_test (SEGFAULT)
        265 - plugins-dockers-animation-timeline_model_test (Failed)
        271 - plugins-generators-kis_seexpr_generator_test (Failed)
        273 - plugins-impex-kis_kra_saver_test (SEGFAULT)

and another run:

         24 - libs-flake-TestPathTool (Failed)
         47 - libs-flake-TestSvgParser (Failed)
         48 - libs-flake-TestSvgParserCloned (Failed)
         49 - libs-flake-TestSvgParserRoundTrip (Failed)
         51 - libs-pigment-TestKoColorSet (ILLEGAL)
        166 - libs-image-kis_walkers_test (Failed)
        167 - libs-image-kis_cage_transform_worker_test (Failed)
        187 - libs-image-tiles3-kis_tiled_data_manager_test (Failed)
        215 - libs-ui-kis_shape_layer_test (Failed)
        229 - libs-ui-kis_shape_controller_test (SEGFAULT)
        230 - libs-ui-kis_dummies_facade_test (SEGFAULT)
        265 - plugins-dockers-animation-timeline_model_test (Failed)
        271 - plugins-generators-kis_seexpr_generator_test (Failed)
        273 - plugins-impex-kis_kra_saver_test (SEGFAULT)