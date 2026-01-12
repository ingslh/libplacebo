cmake_minimum_required(VERSION 3.16)

# Version approximation based on meson.build
set(PL_MAJOR_VER 7)
set(PL_API_VER 358)
set(PL_FIX_VER 0)
set(PL_VERSION "${PL_MAJOR_VER}.${PL_API_VER}.${PL_FIX_VER}")
set(PL_VERSION_PRETTY "v${PL_VERSION}")

project(libplacebo
    VERSION ${PL_VERSION}
    LANGUAGES C CXX
)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Options
option(PL_ENABLE_DEMOS "Enable demos" ON)
option(PL_ENABLE_TESTS "Enable tests" OFF)
option(PL_ENABLE_BENCH "Enable benchmarks" OFF)
option(PL_ENABLE_FUZZ "Enable fuzzers" OFF)

# Feature options (ON/OFF/AUTO logic implemented via standard find_package checks if not explicitly disabled)
option(PL_USE_VULKAN "Enable Vulkan support" ON)
option(PL_USE_OPENGL "Enable OpenGL support" ON)
option(PL_USE_D3D11 "Enable Direct3D 11 support" ON)
option(PL_USE_LCMS "Enable LittleCMS support" ON)
option(PL_USE_DOVI "Enable Dolby Vision support" OFF)
option(PL_USE_LIBDOVI "Enable libdovi support" OFF)
option(PL_USE_XXHASH "Enable xxhash support" ON)
option(PL_USE_UNWIND "Enable libunwind support" ON)
option(PL_USE_SHADERC "Enable shaderc support" ON)
option(PL_USE_GLSLANG "Enable glslang support" ON)

# Compiler flags
if(MSVC)
    add_compile_options(/W3 /wd4244 /wd4267)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS -DWIN32_LEAN_AND_MEAN -DNOMINMAX -D_USE_MATH_DEFINES)
    add_definitions(-D_WIN32_WINNT=0x0601)
else()
    add_compile_options(-Wall -Wextra -Wundef -Wshadow -Wparentheses -Wpointer-arith -fno-math-errno -fno-signed-zeros -fno-trapping-math)
    add_compile_options(-Wno-sign-compare -Wno-unused-parameter -Wno-missing-field-initializers -Wno-type-limits)
    if(CMAKE_C_COMPILER_ID MATCHES "Clang")
        add_compile_options(-Wno-missing-braces)
    endif()
    add_definitions(-D_ISOC99_SOURCE -D_ISOC11_SOURCE -D_GNU_SOURCE)
endif()

# Dependencies
find_package(Python3 REQUIRED COMPONENTS Interpreter)
find_package(Threads REQUIRED)

if(CMAKE_USE_PTHREADS_INIT)
    add_compile_definitions(PL_HAVE_PTHREAD)
endif()

# Math library
if(UNIX)
    set(CMAKE_C_STANDARD_LIBRARIES "${CMAKE_C_STANDARD_LIBRARIES} -lm")
    set(CMAKE_CXX_STANDARD_LIBRARIES "${CMAKE_CXX_STANDARD_LIBRARIES} -lm")
endif()

# Global build definitions
if(PL_ENABLE_TESTS)
    enable_testing()
endif()

add_subdirectory(src)
add_subdirectory(tools)

if(PL_ENABLE_DEMOS)
    add_subdirectory(demos)
endif()

if(WIN32)
    add_subdirectory(win32)
endif()
