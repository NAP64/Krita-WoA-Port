# Krita-WoA-Port
Porting Krita to Windows on Arm64

*Native build for now


## Prepareation

1. Install Visual Studio 2022 v17.14.33. 
2. Get standalone (or install) python3.13:
    https://www.python.org/ftp/python/3.13.13/python-3.13.13-arm64.zip

3. Install StrawberryPerl (I couldn't find an Arm64 alternative, x64 should be fine tho)

4. Patch and compile OpenSSL1.1.1w w/ patch using MSVC. 
    install to the krita-dep-menagement's **_install** dir of choice, change lib\\\*.lib to lib\\\*.dll.a

    Remember to add perl to path from previous step: straberryperl\perl\site\bin, straberryperl\perl\bin

        perl Configure VC-WIN64-ARM --prefix=\<openssl install path\>
        nmake
        nmake test
        nmake install

    Note: msvc /O2 /Gs0 still results in bad assembly as of v17.14.33 and v18.6.2, thus patched
   
5. git, cmake, ninja, llvm as [link](https://docs.krita.org/en/untranslatable_pages/building_krita.html#prerequisites) says

    *You may want to use newer cmake to avoid putting cmake_policy(SET CMP0074 NEW) everywhere (I have omitted them)*

7. Keep following the website and activate PythonEnv and install the other python requirements

(Looks like OpenSSL is the only item left in this section)


## Krita-deps-management

*This will be a wild ride since I've never used CI before. lmk if there are better practise.*

**git checkout transition.now/win-clang21**

After checking out the transition tag, apply the patch first: 

        git apply <path to this repo>\krita-deps-management\krita-deps-management.patch
        git config --local core.autocrlf input
        git apply <path to this repo>\krita-deps-management\pyqt-builder.patch.lf

Run:

        python krita-deps-management\tools\setup-env.py -d -v PythonEnv -p <llvm-mingw-path>\bin\ -p <llvm-mingw-path>\aarch64-w64-mingw32\bin\ -p <ninja-path>
        base-env.bat

Set your _install dir of choice as KDECI_SHARED_INSTALL_PATH

Set environment vars for QT: LLVM_INSTALL_DIR

Set PATH to add llvm\bin, llvm\aarch64-w64-mingw32\bin\, straberryperl\perl\site\bin, straberryperl\perl\bin, ninja, and ccache.

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

You can replace them in _install\lib\site-packages\pyqtbuild\bundle\dlls after kdm compile finishes;

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

w/ Clang22

         52 - libs-flake-TestSvgParser (Failed)                                 *
         53 - libs-flake-TestSvgParserCloned (Failed)                           *
         54 - libs-flake-TestSvgParserRoundTrip (Failed)
         74 - libs-brush-kis_auto_brush_test (Failed)
        156 - libs-image-kis_algebra_2d_test (Failed)                           *
        174 - libs-image-kis_cage_transform_worker_test (Failed)
        179 - libs-image-kis_asl_layer_style_serializer_test (Failed)           *
        199 - libs-image-tiles3-kis_low_memory_tests (Timeout)            should be fine
        225 - libs-ui-kis_shape_layer_test (Failed)
        253 - libs-resources-TestResourceStorage (Failed)
        282 - plugins-generators-seexpr-kis_seexpr_generator_test (Failed)      *
        286 - plugins-impex-tiff-kis_tiff_test (Failed)
        302 - plugins-impex-heif-KisHeifTest (SEGFAULT)

(*labels delta from a x86-64 build, afaik)

We knew about 52, 53, and 282

156 has something to do with Eigen, where a matrix inversion introduced too much error.

I'll look into the new one soon.

Should I do Qt6 as well?