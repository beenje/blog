<!--
.. title: Developing and compiling tango with conda
.. slug: developing-and-compiling-tango-with-conda
.. date: 2022-06-17 20:53:19 UTC+02:00
.. tags: python,conda,linux,Mac,windows,tango
.. category: conda
.. link:
.. description:
.. type: text
-->

[Conda] is a popular package manager that can quickly install binary packages and their dependencies.

Many [Tango] packages are available on [conda-forge]: `pytango`, `tango-test`, `tango-database`, `tango-starter`, `tango-admin`...
Most are still Linux only, but a few are also available for Windows (like `pytango` and `tango-test`).

What many people don't know or don't think about is that conda can also be used to setup a development environment.
The required compilers and build tools, used by [conda-build] to compile and create packages, can be installed as any conda package.

This is something I often do before to create a recipe to check how to compile a software.

## Installing conda/mamba

If you don't have conda already installed, I recommend using [Mambaforge], an alternative to [miniconda].
[Mambaforge] is a minimal installer for conda that also inludes [mamba] and is configured with [conda-forge] as the default and only channel.

If you haven't heard of `mamba` before, check <https://github.com/mamba-org/mamba>. `mamba` is a fast alternative conda client. You can use it as a drop-in replacement for conda and benefit from faster download and dependency solving.

`conda` is also gaining from this development as `libmamba` was integrated in conda 4.12.0: <https://www.anaconda.com/blog/a-faster-conda-for-a-growing-community>. At this time, it's still an experimental feature that needs to be enabled explicitly.

To install `conda` and `mamba` on Unix like platforms:

```bash
curl -L -O "https://github.com/conda-forge/miniforge/releases/latest/download/Mambaforge-$(uname)-$(uname -m).sh"
bash Mambaforge-$(uname)-$(uname -m).sh  -f -b -p ~/mambaforge
~/mambaforge/bin/mamba init
```

On Windows, use the [Mambaforge-Windows-x86_64.exe](https://github.com/conda-forge/miniforge/releases/latest/download/Mambaforge-Windows-x86_64.exe) installer.

The rest of this post will assume you have conda/mamba installed and conda-forge set as the default channel:

```bash
$ conda config --show channels
channels:
  - conda-forge
```

If you are used to `conda` and don't want to install `mamba`, this isn't an issue. Both are interchangeable.
Just run `conda` instead of `mamba`. Out of habit I often still run `conda activate` as this wasn't initially supported by `mamba`, but this isn't the case anymore.

## Compiling cppTango in a conda environment

Let's say you want to compile [cppTango](https://gitlab.com/tango-controls/cppTango) to work on a bug fix or new feature.
To build it on your Linux distribution, you can of course install all the requirements with your OS package manager (apt, yum, dnf...).
Depending of your OS, versions will differ as well as package names between distributions.

Conda provides a solution that is OS independent. Create the following `tango-dev` environment:

```bash
mamba create -y -n tango-dev make cmake cxx-compiler libtool pkg-config jpeg omniorb cppzmq zeromq tango-idl
mamba activate tango-dev
```

To know the list of packages to install, you can refer to the official [cpptango-feedstock](https://github.com/conda-forge/cpptango-feedstock/blob/main/recipe/meta.yaml#L25). I added `jpeg` here as it's a new requirement to build the main branch.

Activating the `tango-dev` environment will export a few variables, like `CC`, `CXX` and `CMAKE_ARGS`.
You should pass the latter to cmake:

```bash
$ echo $CC
/home/vagrant/mambaforge/envs/tango-dev/bin/x86_64-conda-linux-gnu-cc
$ echo $CXX
/home/vagrant/mambaforge/envs/tango-dev/bin/x86_64-conda-linux-gnu-c++
$ echo $CMAKE_ARGS
-DCMAKE_AR=/home/vagrant/mambaforge/envs/tango-dev/bin/x86_64-conda-linux-gnu-ar -DCMAKE_CXX_COMPILER_AR=/home/vagrant/mambaforge/envs/tango-dev/bin/x86_64-conda-linux-gnu-gcc-ar -DCMAKE_C_COMPILER_AR=/home/vagrant/mambaforge/envs/tango-dev/bin/x86_64-conda-linux-gnu-gcc-ar -DCMAKE_RANLIB=/home/vagrant/mambaforge/envs/tango-dev/bin/x86_64-conda-linux-gnu-ranlib -DCMAKE_CXX_COMPILER_RANLIB=/home/vagrant/mambaforge/envs/tango-dev/bin/x86_64-conda-linux-gnu-gcc-ranlib -DCMAKE_C_COMPILER_RANLIB=/home/vagrant/mambaforge/envs/tango-dev/bin/x86_64-conda-linux-gnu-gcc-ranlib -DCMAKE_LINKER=/home/vagrant/mambaforge/envs/tango-dev/bin/x86_64-conda-linux-gnu-ld -DCMAKE_STRIP=/home/vagrant/mambaforge/envs/tango-dev/bin/x86_64-conda-linux-gnu-strip
```

Note that when using [conda-build] even more options are automatically added to `CMAKE_ARGS`, like `CMAKE_INSTALL_PREFIX` and `CMAKE_INSTALL_LIBDIR`.
We need to pass those manually in our case. Use `CONDA_PREFIX` to refer to the current activated environment (this variable is automatically set by conda).

```bash
(tango-dev) vagrant@bullseye:~$ git clone https://gitlab.com/tango-controls/cppTango.git
(tango-dev) vagrant@bullseye:~$ cd cppTango
(tango-dev) vagrant@bullseye:~/cppTango$ cmake $CMAKE_ARGS \
      -DCMAKE_BUILD_TYPE=Debug \
      -DCMAKE_INSTALL_PREFIX="$CONDA_PREFIX" \
      -DCMAKE_INSTALL_LIBDIR=lib \
      -S . -B build
-- The C compiler identification is GNU 10.3.0
-- The CXX compiler identification is GNU 10.3.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /home/vagrant/mambaforge/envs/tango-dev/bin/x86_64-conda-linux-gnu-cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /home/vagrant/mambaforge/envs/tango-dev/bin/x86_64-conda-linux-gnu-c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Found PkgConfig: /home/vagrant/mambaforge/envs/tango-dev/bin/pkg-config (found version "0.29.2")
-- CMake: version 3.23.2
-- Target platform: Linux 64-bit
-- C++ Compiler: GNU with version 10.3.0
-- C Compiler: GNU with version 10.3.0
-- Build type: Debug
-- Environment: TANGO_HOST=
-- Checking for one of the modules 'tangoidl'
-- Checking for one of the modules 'omniORB4'
-- Checking for one of the modules 'omniCOS4'
-- Checking for one of the modules 'omniDynamic4'
-- Checking for one of the modules 'libzmq'
-- Found JPEG: /home/vagrant/mambaforge/envs/tango-dev/lib/libjpeg.so (found version "90")
-- Check if cppzmq is present and recent enough: TRUE
-- Check if libzmq version is >= 4.0.5: TRUE
-- Looking for pthread.h
-- Looking for pthread.h - found
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD - Failed
-- Looking for pthread_create in pthreads
-- Looking for pthread_create in pthreads - not found
-- Looking for pthread_create in pthread
-- Looking for pthread_create in pthread - found
-- Found Threads: TRUE
-- Looking for io.h
-- Looking for io.h - not found
-- Looking for unistd.h
-- Looking for unistd.h - found
-- Could NOT find Doxygen (missing: DOXYGEN_EXECUTABLE)
CMake Warning at cppapi/doxygen/CMakeLists.txt:10 (message):
  Could not find doxygen/dot, therefore the documentation can not be built.


Using OMNIIDL_PATH=
Using IDL=/home/vagrant/mambaforge/envs/tango-dev/include
HOST_NAME=bullseye
-- Configuring done
-- Generating done
-- Build files have been written to: /home/vagrant/cppTango/build
```

Note that if you want to build the documentation, you can install `doxygen` and `graphviz`: `mamba install doxygen graphviz`.

To build: `cmake --build build -j2`

To test: `cmake --build build --target test`

To install: `cmake --build build --target install`

Those commands were run on Debian 11.3 but would work on any Linux distribution.

Let's look at another example with [TangoDatabase].

## Building TangoDatabase with conda

What if you want to work on [TangoDatabase]?
You could build it against the current release of `cppTango`. Check the requirements in the official [tango-database-feedstock](https://github.com/conda-forge/tango-database-feedstock/blob/main/recipe/meta.yaml#L16):

```bash
$ mamba create -y -n tango-stable-dev cmake make cxx-compiler libtool pkg-config cppzmq cpptango mysql-devel omniorb
$ mamba list -n tango-stable-dev | grep cpptango
cpptango                  9.3.5                he78764c_1    conda-forge
```

What if you want to work against `cppTango` main branch? You could of course re-use the previous environment where you compiled it yourself. But there is another option! For development purpose, each commit to `cppTango` main branch,
creates and publishes a conda package to the `tango-controls` channel: <https://anaconda.org/tango-controls/cpptango> (Linux only).
Let's use that package from the `tango-controls/label/dev` channel.

```bash
$ mamba create -y -n tango-main-dev -c tango-controls/label/dev cmake make cxx-compiler libtool pkg-config cppzmq cpptango mysql-devel omniorb
$ mamba list -n tango-main-dev | grep cpptango
cpptango                  9.4.0dev0             g5beab02d    tango-controls/label/dev
```

We can now build `TangoDatabase` in the `tango-main-dev` environment.

```bash
(base) vagrant@bullseye:~$ conda activate tango-main-dev
(tango-main-dev) vagrant@bullseye:~$ git clone https://gitlab.com/tango-controls/TangoDatabase.git
(tango-main-dev) vagrant@bullseye:~$ cd TangoDatabase/
(tango-main-dev) vagrant@bullseye:~/TangoDatabase$ cmake $CMAKE_ARGS \
      -DCMAKE_BUILD_TYPE=Debug \
      -DCMAKE_INSTALL_PREFIX=$CONDA_PREFIX \
      -DMYSQL_INCLUDE_DIR=$CONDA_PREFIX/include/mysql \
      -S . -B build
(tango-main-dev) vagrant@bullseye:~/TangoDatabase$ cmake --build build -j2
(tango-main-dev) vagrant@bullseye:~/TangoDatabase$ ./build/Databaseds --help
main(): arrived
usage :  ./build/Databaseds instance_name [-v[trace level]] [-file=<file_name> | -nodb [-dlist <device name list>] ]
```

Linux is well supported. What about other operating systems?

## Building cppTango on macOS

Yes, Tango on macOS! This still isn't supported officially but [Thomas Juerges](https://gitlab.com/tjuerges) is working on it. At the time of this writing, this hasn't been merged upstream and you need to use the [main-macOS_build branch](https://gitlab.com/tjuerges/cppTango/-/tree/main-macOS_build) from his [repository](https://gitlab.com/tjuerges/cppTango).

Note that this is not an official Tango Controls blessed repository, nor does Tango Controls officially support macOS.

To build this fork, Thomas created a repository with some scripts to help installing all required dependencies using [brew](https://brew.sh): <https://gitlab.com/tjuerges/build_tango>

This works nicely, but we'll look here at an alternative with conda.

Conda provides the clang compilers for macOS but the macOS SDK is still required. Due to its license it can't be installed with conda. If you haven't already done, install the Xcode Command Line Tools by running `xcode-select --install`. All the rest will be installed with conda.

Let's create the same `tango-dev` environment as on Linux:

```bash
mamba create -y -n tango-dev make cmake cxx-compiler libtool pkg-config jpeg omniorb cppzmq zeromq tango-idl
conda activate tango-dev
```

This environment is actually not exacly identical to the Linux one as the `cxx-compiler` meta-package will install `clang` on macOS and `gcc/gxx` on Linux:

```bash
(tango-dev) ➜  ~ $ echo $CC
x86_64-apple-darwin13.4.0-clang
(tango-dev) ➜  ~ $ echo $CXX
x86_64-apple-darwin13.4.0-clang++
```

Compile the `main-macOS_build` branch:

```bash
(tango-dev) ➜  Tango $ git clone https://gitlab.com/tjuerges/cppTango.git
(tango-dev) ➜  Tango $ cd cppTango
(tango-dev) ➜  cppTango git:(main) $ git checkout main-macOS_build
branch 'main-macOS_build' set up to track 'origin/main-macOS_build'.
Switched to a new branch 'main-macOS_build'
(tango-dev) ➜  cppTango git:(main-macOS_build) $ cmake $CMAKE_ARGS \
      -DCMAKE_BUILD_TYPE=Debug \
      -DCMAKE_INSTALL_PREFIX="$CONDA_PREFIX" \
      -DCMAKE_INSTALL_LIBDIR=lib \
      -S . -B build
-- The C compiler identification is Clang 13.0.1
-- The CXX compiler identification is Clang 13.0.1
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /Users/benjaminbertrand/miniconda/envs/tango-dev/bin/x86_64-apple-darwin13.4.0-clang - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /Users/benjaminbertrand/miniconda/envs/tango-dev/bin/x86_64-apple-darwin13.4.0-clang++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Found PkgConfig: /Users/benjaminbertrand/miniconda/envs/tango-dev/bin/pkg-config (found version "0.29.2")
-- CMake: version 3.23.2
-- Target platform: Darwin 64-bit
-- C++ Compiler: Clang with version 13.0.1
-- C Compiler: Clang with version 13.0.1
-- Build type: Debug
-- Environment: TANGO_HOST=
-- Checking for one of the modules 'tangoidl'
-- Checking for one of the modules 'omniORB4'
-- Checking for one of the modules 'omniCOS4'
-- Checking for one of the modules 'omniDynamic4'
-- Checking for one of the modules 'libzmq'
-- Found JPEG: /Users/benjaminbertrand/miniconda/envs/tango-dev/lib/libjpeg.dylib (found version "90")
-- Check if cppzmq is present and recent enough: TRUE
-- Check if libzmq version is >= 4.0.5: TRUE
-- Looking for pthread.h
-- Looking for pthread.h - found
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD - Success
-- Found Threads: TRUE
-- Looking for io.h
-- Looking for io.h - not found
-- Looking for unistd.h
-- Looking for unistd.h - found
-- Could NOT find Doxygen (missing: DOXYGEN_EXECUTABLE)
CMake Warning at cppapi/doxygen/CMakeLists.txt:10 (message):
  Could not find doxygen/dot, therefore the documentation can not be built.


Using OMNIIDL_PATH=
Using IDL=/Users/benjaminbertrand/miniconda/envs/tango-dev/include
HOST_NAME=benjimbp.local
-- Configuring done
-- Generating done
-- Build files have been written to: /Users/benjaminbertrand/Dev/Tango/cppTango/build
```

Build and install as on Linux:

```bash
cmake --build build -j4
cmake --build build --target install
```

If you prefer you can also run `make` directly:

```bash
cd build
make -j4
make install
```

As you can see, the instructions, including how to create the `tango-dev` environment, were the same as on Linux!

We can now build `pytango`. Support was recently added to the `develop_9.4` branch in [MR 459](https://gitlab.com/tango-controls/pytango/-/merge_requests/459).

We first need to add a few extra dependencies required to build `pytango`: `mamba install -y boost numpy six`.
`six` is only required at runtime, not build time, but we add it now as we'll want to run `pytango` to test it.

```bash
cd ..
git clone https://gitlab.com/tango-controls/pytango.git
git checkout -b develop_9.4 origin/develop_9.4
cd pytango
```

You first need to export the `BOOST_PYTHON_LIB` variable based on the installed python version. You can check it by running `python --version`:

```bash
(tango-dev) ➜  pytango git:(develop_9.4) $ python --version
Python 3.10.5

(tango-dev) ➜  pytango git:(develop_9.4) $ export BOOST_PYTHON_LIB=boost_python310
(tango-dev) ➜  pytango git:(develop_9.4) $ python -m pip install --no-binary=:all: --ignore-installed --no-deps .
```

Let's try it:

```bash
(tango-dev) ➜  ~ $ python -c "import tango; print(tango.utils.info())"
PyTango 9.3.4 (9, 3, 4, 'dev', 0)
PyTango compiled with:
    Python : 3.10.5
    Numpy  : 1.22.4
    Tango  : 9.4.0
    Boost  : 1.78.0

PyTango runtime is:
    Python : 3.10.5
    Numpy  : 1.22.4
    Tango  : 9.4.0

PyTango running on:
uname_result(system='Darwin', node='benjimbp.local', release='21.4.0', version='Darwin Kernel Version 21.4.0: Fri Mar 18 00:45:05 PDT 2022; root:xnu-8020.101.4~15/RELEASE_X86_64', machine='x86_64')
```

Hurray!

## Compiling on Windows

To compile on Windows you first need to install the **Build Tools for Visual Studio 2017**. This is the version recommended by conda-forge to create Windows packages. To compile locally, you should be able to use a more recent version.
I described in a previous post how to create a [Windows VM to build conda packages](/posts/how-to-setup-a-windows-vm-to-build-conda-packages>).
You can refer to it for more details.

We will compile TangoTest v3.4 as an example.

```console
(base) Z:\>mamba create -y -n tango-dev cmake cxx-compiler cppzmq cpptango omniorb
(base) Z:\>mamba activate tango-dev
(tango-dev) Z:\>git clone https://gitlab.com/tango-controls/TangoTest.git
(tango-dev) Z:\>cd TangoTest
(tango-dev) Z:\>git checkout 3.4
```

As there is no `pkg-config` on Windows, there are a few extra variables to pass to cmake. Let's create a small `bld.bat` script to make that easier:

```bat
setlocal EnableDelayedExpansion

set LIBRARY_PREFIX=%CONDA_PREFIX%\Library
set LIBRARY_LIB=%CONDA_PREFIX%\Library\lib
set LIBRARY_INC=%CONDA_PREFIX%\Library\include

cmake -G "NMake Makefiles" ^
      -DCMAKE_CXX_FLAGS="-DLOG4TANGO_HAS_DLL -DTANGO_HAS_DLL" ^
      -DCMAKE_INSTALL_PREFIX:PATH="%LIBRARY_PREFIX%" ^
      -DCMAKE_PREFIX_PATH:PATH="%LIBRARY_PREFIX%" ^
      -DTANGO_PKG_LIBRARY_DIRS:PATH="%LIBRARY_LIB%" ^
      -DTANGO_PKG_INCLUDE_DIRS:PATH="%LIBRARY_INC%" ^
      -DTANGO_PKG_LIBRARIES="tango;omniORB4_rt;omniDynamic4_rt;COS4_rt;omnithread_rt;libzmq-mt-4_3_4;comctl32;wsock32;Ws2_32" ^
      -S . -B build
```

This script is based on the one from the [official recipe](https://github.com/conda-forge/tango-test-feedstock/blob/main/recipe/bld.bat).
The variables `LIBRARY_PREFIX`, `LIBRARY_LIB` and `LIBRARY_INC` are automatically set by `conda-build`. When compiling manually we define them using `CONDA_PREFIX`.

Run `bld.bat`:

```console
(tango-dev) Z:\TangoTest>bld.bat
(tango-dev) Z:\TangoTest>setlocal EnableDelayedExpansion
(tango-dev) Z:\TangoTest>set LIBRARY_PREFIX=C:\Users\beenj\mambaforge\envs\tango-dev\Library
(tango-dev) Z:\TangoTest>set LIBRARY_LIB=C:\Users\beenj\mambaforge\envs\tango-dev\Library\lib
(tango-dev) Z:\TangoTest>set LIBRARY_INC=C:\Users\beenj\mambaforge\envs\tango-dev\Library\include
(tango-dev) Z:\TangoTest>cmake -G "NMake Makefiles"       -DCMAKE_CXX_FLAGS="-DLOG4TANGO_HAS_DLL -DTANGO_HAS_DLL"       -DCMAKE_INSTALL_PREFIX:PATH="C:\Users\beenj\mambaforge\envs\tango-dev\Library"       -DCMAKE_PREFIX_PATH:PATH="C:\Users\beenj\mambaforge\envs\tango-dev\Library"       -DTANGO_PKG_LIBRARY_DIRS:PATH="C:\Users\beenj\mambaforge\envs\tango-dev\Library\lib"       -DTANGO_PKG_INCLUDE_DIRS:PATH="C:\Users\beenj\mambaforge\envs\tango-dev\Library\include"       -DTANGO_PKG_LIBRARIES="tango;omniORB4_rt;omniDynamic4_rt;COS4_rt;omnithread_rt;libzmq-mt-4_3_4;comctl32;wsock32;Ws2_32"       -S . -B build
-- The CXX compiler identification is MSVC 19.16.27045.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: C:/Program Files (x86)/Microsoft Visual Studio/2017/BuildTools/VC/Tools/MSVC/14.16.27023/bin/Hostx64/x64/cl.exe - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Could NOT find PkgConfig (missing: PKG_CONFIG_EXECUTABLE)
-- Configuring done
-- Generating done
-- Build files have been written to: Z:/TangoTest/build
```

We can now compile:

```console
(tango-dev) Z:\TangoTest>cmake --build build
[ 16%] Building CXX object CMakeFiles/TangoTest.dir/ClassFactory.cpp.obj
ClassFactory.cpp
Z:\TangoTest\ClassFactory.cpp(57): fatal error C1083: Cannot open include file: 'tango.h': No such file or directory
NMAKE : fatal error U1077: 'C:\Users\beenj\mambaforge\envs\tango-dev\Library\bin\cmake.exe' : return code '0x2'
Stop.
```

Something went wrong...
`tango.h` wans't found. This is because includes were recently moved in Windows. See [MR924](https://gitlab.com/tango-controls/cppTango/-/merge_requests/924). There is actually a patch for that in the [recipe](https://github.com/conda-forge/tango-test-feedstock/blob/main/recipe/tango-include-dirs.patch).
Update the `CMakeLists.txt` to add `${TANGO_PKG_INCLUDE_DIRS}/tango` to the include directories.

```diff
(tango-dev) Z:\TangoTest>git diff
diff --git a/CMakeLists.txt b/CMakeLists.txt
index 5f96806..f36f348 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -26,7 +26,7 @@ set(HEADERS
 link_directories(${TANGO_PKG_LIBRARY_DIRS})

 add_executable(${PROJECT_NAME} ${SOURCES} ${HEADERS})
-target_include_directories(${PROJECT_NAME} PUBLIC ${CMAKE_CURRENT_SOURCE_DIR} ${TANGO_PKG_INCLUDE_DIRS})
+target_include_directories(${PROJECT_NAME} PUBLIC ${CMAKE_CURRENT_SOURCE_DIR} ${TANGO_PKG_INCLUDE_DIRS} ${TANGO_PKG_INCLUDE_DIRS}/tango)
 target_compile_options(${PROJECT_NAME} PUBLIC ${TANGO_PKG_CFLAGS_OTHER})
 target_link_libraries(${PROJECT_NAME} PUBLIC ${TANGO_PKG_LIBRARIES})
```

Run `cmake` again:

```console
(tango-dev) Z:\TangoTest>bld.bat
(tango-dev) Z:\TangoTest>cmake --build build

[100%] Linking CXX executable TangoTest.exe
[100%] Built target TangoTest
(tango-dev) Z:\TangoTest>cd build
(tango-dev) Z:\TangoTest\build>TangoTest.exe --help
usage :  TangoTest.exe instance_name [-v[trace level]] [-file=<file_name> | -nodb [-dlist <device name list>] ]
```

Build successful on Windows!

## Conclusion

[Conda] gives a reliable way to install binary packages whatever the operating system.
The long list of packages maintained by [conda-forge] includes compilers and build tools,
allowing to easily setup a development environment which is OS independent.
You can rely on modern versions of those tools even on an older OS.

Naturally conda main goal is to avoid having you to compile by installing pre-build packages :-).
This is for development and testing only. Compile only what you need.
And if something is missing, submit it to conda-forge [staged-recipes](https://github.com/conda-forge/staged-recipes)!

[Conda] is of course just an alternative.
Sometimes you do have to compile natively to check a platform compatibility.
Use the solution that works for you!

[conda]: https://docs.conda.io/en/latest/
[conda-build]: https://docs.conda.io/projects/conda-build/en/latest/index.html
[mamba]: https://mamba.readthedocs.io/en/latest/index.html
[Mambaforge]: https://github.com/conda-forge/miniforge
[miniconda]: https://docs.conda.io/en/latest/miniconda.html
[conda-forge]: https://github.com/conda-forge
[Tango]: https://www.tango-controls.org
[TangoDatabase]: https://gitlab.com/tango-controls/TangoDatabase
