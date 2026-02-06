# Krita-WoA-Port
Porting Krita to Windows on Arm64

*Native build for now


## Prepareation

1. Install Visual Studio 2022 v17.14.25. Run following ilnes in this section in a prompt with VS developer environemnt
2. Compile and Install Python 3.10, along with embeded version. collect **python-3.10-embed-arm64.zip**

        PCbuild\build.bat -p arm64
        python.bat PC\layout -b PCbuild\arm64 -s . -t PCbuild\obj\310arm64_Release\msi_python\zip_arm64 --zip python-3.10-embed-arm64.zip --precompile --zip-lib --include-underpth --include-stable --flat-dlls
        python.bat PC\layout --precompile --preset-default --copy \<python install path\>

3. Install StrawberryPerl (I couldn't find an Arm64 alternative, x64 should be fine tho)

4. Patch and compile OpenSSL1.1.1w w/ patch using MSVC. 
    install to the krita-dep-menagement's **_install** dir of choice, change lib\\\*.lib to lib\\\*.dll.a

    Remember to add perl to path from previous step: straberryperl\perl\site\bin, straberryperl\perl\bin

        perl Configure VC-WIN64-ARM --prefix=\<openssl install path\>
        nmake
        nmake test
        nmake install

    Note: msvc /O2 /Gs0 still results in bad assembly as of v17.14.25 and v18.1.0, thus patched
   
5. git, cmake, ninja, llvm as [link](https://docs.krita.org/en/untranslatable_pages/building_krita.html#prerequisites) says

    *You may want to use newer cmake to avoid putting cmake_policy(SET CMP0074 NEW) everywhere (I have omitted them)*

6. Download and install ccache. I have a directory with ninja and ccache executables added to path .
7. Keep following the website and activate PythonEnv and install the other python requirements


## Krita-deps-management

*This will be a wild ride since I've never used CI before. lmk if there are better practise.*

Apply the patch first: 

        git apply <path to this repo>\krita-deps-management\krita-deps-management.patch
        git config --local core.autocrlf input
        git apply <path to this repo>\krita-deps-management\pyqt-builder.patch.lf

Run:

        python krita-deps-management\tools\setup-env.py -d -v PythonEnv -p <llvm-mingw-path>\bin\ -p <llvm-mingw-path>\aarch64-w64-mingw32\bin\ -p <ninja-path>
        base-env.bat

Set your _install dir of choice as KDECI_SHARED_INSTALL_PATH

Set environment vars for QT: LLVM_INSTALL_DIR

Set PATH to add llvm\bin, llvm\aarch64-w64-mingw32\bin\, straberryperl\perl\site\bin, straberryperl\perl\bin, ninja, and ccache.

Replace line 107 in krita-deps-management\ext_python\CMakeLists.txt to point to the python-3.10-embed-arm64.zip  when you compile python (change the direction of slashes).

Run the command to run all builds: 

        python -u ci-utilities/seed-package-registry.py --seed-file latest/krita-deps.yml --platform Windows/Qt5/Shared --skip-dependencies-fetch

Command to compile one dependency, for reference: 

        python -u ..\ci-utilities\run-ci-build.py --project ext_<name> --branch master --platform Windows/Qt5/Shared --only-build --fail-on-leaked-stage-files --skip-dependencies-fetch

There are a few dependencies that won't compile for Windows on Arm, commented out in latest/krita.dep, check diff to find out.
Among them, brotli can compile w/o libjxl or highway (patched).

I'm trying if newer tools can help, so Boost is now 1.87 instead of 1.80. To revert simply change the numbers and enable the patches.

*Note: freetype compiles but seems to have issue, but required. Leaving as-is.*

The edit in ext_sip is to workaround a config error in a later stage.
I'm too lazy to locate the config source.

The edit in ext_pyqt_builder is to have arm64 libs alongside of x64 ones. 
You don't have to use my binaries:

Take the openssl dlls from the openssl step above,
and pyqt-builder has arm64 msvc dlls in their newer versions (i.e. 1.19.1).
I'm using these, too (although I'm not sure if they will be used and where).

As of July 2025, llvm 20 and 21 updated ptherad_time.h. I removed mlt's src\win32\win32.c nanosleep function for it to compile (either llvm 20 ships with one or windows has one.)

As of Dec 2025, llvm21 updated include\c++\v1\\__type_traits\is_integral.h. Move the clang-format block (with value = 1 templates) before the __has_builtin(__is_integral) macro if,worked for me.

## Compile Krita ##

Set PATH (from previous step) to add \<deps _install dir\>\bin and \<deps _install dir\>\lib

Set PYTHONPATH to \<deps _install dir\>\lib\site-packages

Point -DMAKE_INSTALL_PREFIX to your _install dir of choice, Krita will compile normally with one simple fix: Copy krita.ico and kritafile.ico from a Krita installation to b_krita/krita.
This is needed since icoutils won't compile for aarch64.

To package:

Apply the patch to krita src:

        git apply <path to this repo>\krita.patch

Run the following, a zip with your name of choice will be created:

        python <krita src>\packaging\windows\package-complete.py --src-dir <krita src>\ --deps-install-dir <deps _install dir> --krita-install-dir <deps _install dir> --package-name <krita name of choice>

## Testing ##

w/ Clang18:

         24 - libs-flake-TestPathTool (Failed)
         47 - libs-flake-TestSvgParser (Failed)
         48 - libs-flake-TestSvgParserCloned (Failed)
         49 - libs-flake-TestSvgParserRoundTrip (Failed)
         51 - libs-pigment-TestKoColorSet (ILLEGAL)
         68 - libs-brush-kis_auto_brush_test (Failed)
        163 - libs-image-kis_onion_skin_compositor_test (Failed)
        165 - libs-image-kis_image_animation_interface_test (Failed)
        166 - libs-image-kis_walkers_test (Failed)
        167 - libs-image-kis_cage_transform_worker_test (Failed)
        187 - libs-image-tiles3-kis_tiled_data_manager_test (Failed)
        188 - libs-image-tiles3-kis_low_memory_tests (Timeout)
        215 - libs-ui-kis_shape_layer_test (Failed)
        229 - libs-ui-kis_shape_controller_test (SEGFAULT)
        230 - libs-ui-kis_dummies_facade_test (SEGFAULT)
        265 - plugins-dockers-animation-timeline_model_test (Failed)
        271 - plugins-generators-kis_seexpr_generator_test (Failed)
        273 - plugins-impex-kis_kra_saver_test (SEGFAULT)
        299 - plugins-tooltransform-test_animated_transform_parameters (Failed)

qt5-base has many failed tests by itself, even with x64 build. On Windows on Arm, some tests fail, but not with the debug test exe. Not sure why.

w/ Clang21

         24 - libs-flake-TestPathTool (Failed)
         47 - libs-flake-TestSvgParser (Failed)
         48 - libs-flake-TestSvgParserCloned (Failed)
         49 - libs-flake-TestSvgParserRoundTrip (Failed)
         51 - libs-pigment-TestKoColorSet (ILLEGAL)
         68 - libs-brush-kis_auto_brush_test (Failed)
        163 - libs-image-kis_onion_skin_compositor_test (Failed)
        165 - libs-image-kis_image_animation_interface_test (Failed)
        166 - libs-image-kis_walkers_test (Failed)
        167 - libs-image-kis_cage_transform_worker_test (Failed)
        187 - libs-image-tiles3-kis_tiled_data_manager_test (Failed)
        229 - libs-ui-kis_shape_controller_test (SEGFAULT)
        230 - libs-ui-kis_dummies_facade_test (SEGFAULT)
        265 - plugins-dockers-animation-timeline_model_test (Failed)
        271 - plugins-generators-kis_seexpr_generator_test (Failed)
        273 - plugins-impex-kis_kra_saver_test (SEGFAULT)
        299 - plugins-tooltransform-test_animated_transform_parameters (Failed)