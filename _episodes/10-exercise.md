---
layout: episode
title: "Exercise: CMake-ify an example project"
teaching: 20
exercises: 35
questions:
  - What is a typical layout for a CMake framework?
  - How can we build static and shared libraries?
  - How can we get version information into the binary output for reproducibility?
  - How does testing work in CMake?
  - How can we create an installer and packager?
keypoints:
  - Structure your CMake project in a modular way.
  - You always want to print version information in your program output for reproducibility.
---

## Creating a CMake framework for an example project

In this exercise we will CMake-ify an [example
project](https://github.com/bast/calculator).  This exercise can be interesting
for people who use Makefiles or Autotools.

On the way we will experiment with some CMake features.

The source code and unit tests are there:

```
.
├── LICENSE.md
├── README.md
├── src
│   ├── add.f90
│   ├── calculator.h
│   ├── main.cpp
│   └── subtract.f90
└── test
    ├── calculator.cpp
    └── main.cpp
```

You can also browse them [on the web](https://github.com/bast/calculator).

This is how it looks when we run the code:

```shell
$ ./bin/calculator.x

2 + 3 = 5
2 - 3 = -1
```

### Tasks

- Build a shared library.
- Build and link the main program.
- Build the unit tests and link against [Google Test](https://github.com/google/googletest).
- Define a version number inside CMake and print it to the output of the executable.
- Print the Git hash to the output of the executable.
- Create an installer so the program can be installed properly (GNU standards).
- Create a DEB or RPM package (if relevant for your distribution).


### Solution

You can find a solution on the [solution branch](https://github.com/bast/calculator/tree/solution).

---

## Building the sources

First we create a file called `CMakeLists.txt` containing:

```cmake
# stop if CMake version is below 3.0
cmake_minimum_required(VERSION 3.0 FATAL_ERROR)

# project name and supported languages
project(calculator CXX Fortran)

# specify where to place binaries and libraries
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY
    ${CMAKE_BINARY_DIR}/bin
    )
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY
    ${CMAKE_BINARY_DIR}/lib
    )

# tell CMake where to find *.cmake module files
set(CMAKE_MODULE_PATH
    ${CMAKE_MODULE_PATH}
    ${PROJECT_SOURCE_DIR}/cmake
    )

# process src/CMakeLists.txt
add_subdirectory(src)
```

We also create a file `src/CMakeLists.txt` containing:

```cmake
add_executable(
    calculator.x
    add.f90
    subtract.f90
    calculator.h
    main.cpp
    )
```

```shell
$ mkdir build
$ cd build
$ cmake ..
$ make

Scanning dependencies of target calculator.x
[ 25%] Building Fortran object src/CMakeFiles/calculator.x.dir/add.f90.o
[ 50%] Building Fortran object src/CMakeFiles/calculator.x.dir/subtract.f90.o
[ 75%] Building CXX object src/CMakeFiles/calculator.x.dir/main.cpp.o
[100%] Linking CXX executable ../bin/calculator.x
[100%] Built target calculator.x
```

Let us rewrite the `src/CMakeLists.txt` a bit to isolate the library, we also ask for a shared library:

```cmake
add_library(
    calculator
    SHARED
    add.f90
    subtract.f90
    calculator.h
    )

add_executable(
    calculator.x
    main.cpp
    )

target_link_libraries(
    calculator.x
    calculator
    )
```

And recompile:

```shell
$ make

-- Configuring done
-- Generating done
-- Build files have been written to: /home/bast/tmp/calculator/build
Scanning dependencies of target calculator
[ 20%] Building Fortran object src/CMakeFiles/calculator.dir/add.f90.o
[ 40%] Building Fortran object src/CMakeFiles/calculator.dir/subtract.f90.o
[ 60%] Linking Fortran shared library ../lib/libcalculator.so
[ 60%] Built target calculator
Scanning dependencies of target calculator.x
[ 80%] Building CXX object src/CMakeFiles/calculator.x.dir/main.cpp.o
[100%] Linking CXX executable ../bin/calculator.x
[100%] Built target calculator.x
```

We have now managed to compile a binary to `build/bin/calculator.x` and a shared
library to `build/lib/libcalculator.so`.

---

## Introduce a tweak for a specific architecture

It turns out that we will need the following tweak for Mac to avoid a warning:

```cmake
# workaround for CMP0042 warning on Mac
if(CMAKE_SYSTEM_NAME MATCHES "Darwin")
    if(NOT DEFINED CMAKE_MACOSX_RPATH)
        set(CMAKE_MACOSX_RPATH ON)
    endif()
endif()
```

We could place it into the main `CMakeLists.txt` but we will rather place this
into a separate file `cmake/arch.cmake` (name is our choice) and include it in
the main `CMakeLists.txt`. We do this to have a cleaner and more modular
structure of the CMake code.

Save the following code into `cmake/arch.cmake`:

```cmake
message(STATUS "My system is ${CMAKE_SYSTEM_NAME} and my processor is ${CMAKE_HOST_SYSTEM_PROCESSOR}")

# workaround for CMP0042 warning on Mac
if(CMAKE_SYSTEM_NAME MATCHES "Darwin")
    if(NOT DEFINED CMAKE_MACOSX_RPATH)
        set(CMAKE_MACOSX_RPATH ON)
    endif()
endif()
```

Now we need to include this in the main `CMakeLists.txt`:

```cmake
# ... rest of CMakeLists.txt

# tell CMake where to find *.cmake module files
set(CMAKE_MODULE_PATH
    ${CMAKE_MODULE_PATH}
    ${PROJECT_SOURCE_DIR}/cmake
    )

# include cmake/arch.cmake
include(arch)

# process src/CMakeLists.txt
add_subdirectory(src)
```

Then try it out. On Mac, the warning should be gone. On my system I get:

```shell
$ make

-- My system is Linux and my processor is x86_64
...
```

---

## Build the unit tests and link against [Google Test](https://github.com/google/googletest)

Save this to `cmake/test.cmake`:

```cmake
    include(ExternalProject)

    ExternalProject_Add(
        gtest
        PREFIX "${PROJECT_BINARY_DIR}/gtest"
        GIT_REPOSITORY https://github.com/google/googletest.git
        GIT_TAG master
        INSTALL_COMMAND true  # currently no install command
        )

    add_executable(
        unit_tests
        test/main.cpp
        test/calculator.cpp
        )

    target_link_libraries(
        unit_tests
        ${PROJECT_BINARY_DIR}/gtest/src/gtest-build/googlemock/gtest/libgtest.a
        calculator
        pthread
        )

    target_include_directories(
        unit_tests
        PRIVATE
        ${PROJECT_BINARY_DIR}/gtest/src/gtest/googletest/include
        ${PROJECT_SOURCE_DIR}/src
        )

    add_dependencies(unit_tests gtest)

    include(CTest)
    enable_testing()

    add_test(unit ${PROJECT_BINARY_DIR}/bin/unit_tests)
```

Also include this file in the main `CMakeLists.txt`:

```cmake
# ... rest of CMakeLists.txt

include(arch)

# process src/CMakeLists.txt
add_subdirectory(src)

include(test)  # we added this line
```

Now try:

```shell
$ make
$ make test
$ ./bin/unit_tests
```

What is the difference between `make test` and `./bin/unit_tests`?

We will enhance this a bit: We add an option and an if-check to be able to toggle compilation of unit tests on or off:

```cmake
option(ENABLE_UNIT_TESTS "Enable unit tests" ON)
message(STATUS "Enable testing: ${ENABLE_UNIT_TESTS}")

if(ENABLE_UNIT_TESTS)
    include(ExternalProject)

    ExternalProject_Add(
        gtest
        PREFIX "${PROJECT_BINARY_DIR}/gtest"
        GIT_REPOSITORY https://github.com/google/googletest.git
        GIT_TAG master
        INSTALL_COMMAND true  # currently no install command
        )

    add_executable(
        unit_tests
        test/main.cpp
        test/calculator.cpp
        )

    target_link_libraries(
        unit_tests
        ${PROJECT_BINARY_DIR}/gtest/src/gtest-build/googlemock/gtest/libgtest.a
        calculator
        pthread
        )

    target_include_directories(
        unit_tests
        PRIVATE
        ${PROJECT_BINARY_DIR}/gtest/src/gtest/googletest/include
        ${PROJECT_SOURCE_DIR}/src
        )

    add_dependencies(unit_tests gtest)

    include(CTest)
    enable_testing()

    add_test(unit ${PROJECT_BINARY_DIR}/bin/unit_tests)
endif()
```

Having the option we could now toggle the unit testing off:

```shell
$ cd build
$ cmake -DENABLE_UNIT_TESTS=OFF ..
$ make
```

---

## Define a version number inside CMake and print it to the output of the executable

Create a file `cmake/version.h.in`:

```shell
#define VERSION_MAJOR @VERSION_MAJOR@
#define VERSION_MINOR @VERSION_MINOR@
#define VERSION_PATCH @VERSION_PATCH@
```

Then, create a file `cmake/version.cmake`:

```cmake
# version of our project
set(VERSION_MAJOR 1)
set(VERSION_MINOR 0)
set(VERSION_PATCH 0)

# generate file version.h based on version.h.in
configure_file(
    ${PROJECT_SOURCE_DIR}/cmake/version.h.in
    ${PROJECT_BINARY_DIR}/generated/version.h
    @ONLY
    )
```

Also include it in the main `CMakeLists.txt`.

We also need to add

```cmake
target_include_directories(
    calculator.x
    PRIVATE
    ${PROJECT_BINARY_DIR}/generated
    )
```

to `src/CMakeLists.txt` (why?).

Include `version.h` in `src/main.cpp` and try to print the code version.

---

## Print the Git hash to the output of the executable

For this we enhance `cmake/version.h.in`:

```shell
#define VERSION_MAJOR @VERSION_MAJOR@
#define VERSION_MINOR @VERSION_MINOR@
#define VERSION_PATCH @VERSION_PATCH@
#define GIT_HASH "@GIT_HASH@"
```

As well as `cmake/version.cmake`:

```cmake
# version of our project
set(VERSION_MAJOR 1)
set(VERSION_MINOR 0)
set(VERSION_PATCH 0)

# in case Git is not available, we default to "unknown"
set(GIT_HASH "unknown")

# find Git and if available set GIT_HASH variable
find_package(Git)
if(GIT_FOUND)
    execute_process(
        COMMAND ${GIT_EXECUTABLE} --no-pager show -s --pretty=format:%h -n 1
        OUTPUT_VARIABLE GIT_HASH
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
        )
endif()

# generate file version.h based on version.h.in
configure_file(
    ${PROJECT_SOURCE_DIR}/cmake/version.h.in
    ${PROJECT_BINARY_DIR}/generated/version.h
    @ONLY
    )
```

Try to now print the configure-time Git hash in `src/main.cpp`.

Homework: The above solution records the configure-time Git hash. How would you
record the Git hash at build time?

---

## Create an installer so the program can be installed properly (GNU standards)

Append the following to `src/CMakeLists.txt`:

```cmake
install(
    TARGETS calculator.x
    RUNTIME DESTINATION bin
    )

install(
    TARGETS calculator
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    )
```

Then try the following:

```shell
$ mkdir build
$ cd build
$ cmake -DCMAKE_INSTALL_PREFIX=/tmp/cmake-example
$ make
$ make test
$ make install
```

---

## Create a DEB or RPM package (if relevant for your distribution)

Include a `cmake/packager.cmake` containing:

```cmake
# change this later to a real person
set(CPACK_PACKAGE_CONTACT "Slim Shady")

include(CPack)
```

Then run:

```cmake
$ cpack -G DEB
```

or:

```cmake
$ cpack -G RPM
```
