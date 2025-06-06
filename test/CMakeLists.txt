############################################################################
# Copyright (c) Johan Mabille, Sylvain Corlay, Wolf Vollprecht and         #
# Martin Renou                                                             #
# Copyright (c) QuantStack                                                 #
# Copyright (c) Serge Guelton                                              #
#                                                                          #
# Distributed under the terms of the BSD 3-Clause License.                 #
#                                                                          #
# The full license is in the file LICENSE, distributed with this software. #
############################################################################

cmake_minimum_required(VERSION 3.8)

if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    project(xsimd-test)

    enable_testing()

    find_package(xsimd REQUIRED CONFIG)
    set(XSIMD_INCLUDE_DIR ${xsimd_INCLUDE_DIRS})
endif ()

if(NOT CMAKE_BUILD_TYPE)
    message(STATUS "Setting tests build type to Release")
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
else()
    message(STATUS "Tests build type is ${CMAKE_BUILD_TYPE}")
endif()

include(CheckCXXCompilerFlag)

string(TOUPPER "${CMAKE_BUILD_TYPE}" U_CMAKE_BUILD_TYPE)

OPTION(XSIMD_ENABLE_WERROR "Turn on -Werror" OFF)



################
# ARM SETTINGS #
################

OPTION(CROSS_COMPILE_ARM "cross compile for ARM targets" OFF)

# Note: to compile on ARM (or cross compile), you may need to add the following:
# -DTARGET_ARCH="armv8-a -mfpu=neon -mfloat-abi=softfp -target arm-linux-gnueabi"
set(TARGET_ARCH "native" CACHE STRING "Target architecture arguments")

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Intel")
    if (NOT WIN32 AND NOT ANDROID)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wunused-parameter -Wextra -Wreorder")
        if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
          set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wshorten-64-to-32")
      endif()
        # Users may override the c++ standard:
        if(NOT DEFINED CMAKE_CXX_STANDARD OR "${CMAKE_CXX_STANDARD}" STREQUAL "")
            if (ENABLE_XTL_COMPLEX)
                CHECK_CXX_COMPILER_FLAG("-std=c++17" HAS_CPP17_FLAG)
                if (NOT HAS_CPP17_FLAG)
                    message(FATAL_ERROR "Unsupported compiler -- xsimd requires C++17 support when xtl complex support is enabled")
                endif()
                set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17")
            else()
                CHECK_CXX_COMPILER_FLAG("-std=c++11" HAS_CPP11_FLAG)
                if (NOT HAS_CPP11_FLAG)
                    message(FATAL_ERROR "Unsupported compiler -- xsimd requires C++11 support!")
                else()
                    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
                endif()
            endif()
        endif()

        if (NOT CROSS_COMPILE_ARM)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -fPIC")
        endif()
    endif()

    if (ANDROID)
        # Nothing to do here, we assume the cmake Android NDK toolchain sets the
        # correct options for arm and neon.
    elseif (CROSS_COMPILE_ARM)
        # We're cross-compiling with clang++ on Azure Pipelines, this is all pretty specific and just for testing
        set(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS)
        set(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS)
        set(CMAKE_THREAD_LIBS_INIT)

        set(CMAKE_SYSTEM_PROCESSOR arm)
        set(CMAKE_C_COMPILER_TARGET arm-linux-gnueabi)
        set(CMAKE_CXX_COMPILER_TARGET arm-linux-gnueabi)

        set(ARM_ARCH_DIRECTORY "arm-linux-gnueabi" CACHE STRING "ARM arch header dir")
        set(ARM_GCC_VER "4.7.3" CACHE STRING "ARM GCC header dir")
        include_directories(/usr/${ARM_ARCH_DIRECTORY}/include/c++/${ARM_GCC_VER}/${ARM_ARCH_DIRECTORY}/)
        include_directories(/usr/${ARM_ARCH_DIRECTORY}/include/c++/${ARM_GCC_VER}/)
        include_directories(/usr/${ARM_ARCH_DIRECTORY}/include/)
        if(NOT CMAKE_CXX_FLAGS MATCHES "-march")
            message(STATUS "SETTING ARCH TO ${TARGET_ARCH}")
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=${TARGET_ARCH}")
        endif()
        if(ARM_ARCH_DIRECTORY MATCHES "arm-linux-gnueabi")
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mfpu=neon -mfloat-abi=softfp -target arm-linux-gnueabi")
        else ()
            # delegating to gcc here
        endif()
        message(STATUS "CMAKE_CXX_FLAGS: ${CMAKE_CXX_FLAGS}")
        message(STATUS "CMAKE_CXX_LINK_EXECUTABLE: ${CMAKE_CXX_LINK_EXECUTABLE}")
    elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "^ppc64"  OR ${CMAKE_SYSTEM_PROCESSOR} MATCHES "aarch64")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mcpu=${TARGET_ARCH} -mtune=${TARGET_ARCH}")
    elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "riscv64")
        # Nothing specific
    elseif(NOT WIN32 AND NOT EMSCRIPTEN)
        if(NOT CMAKE_CXX_FLAGS MATCHES "-march" AND NOT CMAKE_CXX_FLAGS MATCHES "-arch" AND NOT CMAKE_OSX_ARCHITECTURES)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=${TARGET_ARCH}")
        endif()
    endif()
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHsc /MP /bigobj")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4244 /wd4267 /wd4005 /wd4146 /wd4800")
    set(CMAKE_EXE_LINKER_FLAGS /MANIFEST:NO)
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES Clang AND MSVC AND WIN32) # We are using clang-cl
    add_compile_options(/EHsc /bigobj)
    set(CMAKE_EXE_LINKER_FLAGS /MANIFEST:NO)
endif()

set(XSIMD_TESTS
    main.cpp
    test_api.cpp
    test_arch.cpp
    test_basic_math.cpp
    test_batch.cpp
    test_batch_bool.cpp
    test_batch_cast.cpp
    test_batch_complex.cpp
    test_batch_float.cpp
    test_batch_int.cpp
    test_bitwise_cast.cpp
    test_batch_constant.cpp
    test_batch_manip.cpp
    test_complex_exponential.cpp
    test_complex_hyperbolic.cpp
    test_complex_power.cpp
    test_complex_trigonometric.cpp
    test_conversion.cpp
    test_custom_default_arch.cpp
    test_error_gamma.cpp
    test_explicit_batch_instantiation.cpp
    test_exponential.cpp
    test_extract_pair.cpp
    test_fp_manipulation.cpp
    test_hyperbolic.cpp
    test_load_store.cpp
    test_memory.cpp
    test_poly_evaluation.cpp
    test_power.cpp
    test_rounding.cpp
    test_select.cpp
    test_shuffle.cpp
    test_sum.cpp
    test_traits.cpp
    test_trigonometric.cpp
    test_xsimd_api.cpp
    test_utils.hpp
)

if(NOT MSVC)
    list(APPEND XSIMD_TESTS test_gnu_source.cpp)
endif()

add_executable(test_xsimd ${XSIMD_TESTS} ${XSIMD_HEADERS})
target_include_directories(test_xsimd PRIVATE ${XSIMD_INCLUDE_DIR})

option(DOWNLOAD_DOCTEST OFF)
find_package(doctest QUIET)
if (doctest_FOUND)
    set(DOCTEST_MINIMAL_VERSION 2.4.9)
    if (doctest_VERSION VERSION_LESS DOCTEST_MINIMAL_VERSION)
        message(FATAL_ERROR "Requires doctest >= ${DOCTEST_MINIMAL_VERSION}")
    endif()
    target_link_libraries(test_xsimd PRIVATE doctest::doctest)
elseif(DOWNLOAD_DOCTEST)
    file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/doctest")
    file(DOWNLOAD
        "https://github.com/doctest/doctest/releases/download/v2.4.9/doctest.h"
        "${CMAKE_CURRENT_BINARY_DIR}/doctest/doctest.h"
        STATUS DOWNLOAD_DOCTEST_STATUS)
    list(GET DOWNLOAD_DOCTEST_STATUS 0 DOWNLOAD_DOCTEST_STATUS_CODE)
    list(GET DOWNLOAD_DOCTEST_STATUS 1 DOWNLOAD_DOCTEST_ERROR_MESSAGE)
    if(${DOWNLOAD_DOCTEST_STATUS_CODE} EQUAL 0)
        message(STATUS "Successfully downloaded doctest.h")
    else()
        message(FATAL_ERROR "Error occurred during download of doctest: ${DOWNLOAD_DOCTEST_ERROR_MESSAGE}")
    endif()
    target_include_directories(test_xsimd PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
else()
    message(FATAL_ERROR "
    Cannot find required doctest component.
    Please either set CMAKE_PREFIX_PATH to the location of doctestConfig.cmake,
    or set DOWNLOAD_DOCTEST=ON")
endif()

if(ENABLE_XTL_COMPLEX)
  add_compile_definitions(XSIMD_ENABLE_XTL_COMPLEX=1)
  target_include_directories(test_xsimd PRIVATE ${xtl_INCLUDE_DIRS})
endif()

add_test(NAME test_xsimd COMMAND test_xsimd)

if (CROSS_COMPILE_ARM)
    add_custom_target(xtest COMMAND qemu-arm -L /usr/arm-linux-gnueabi/ test_xsimd DEPENDS test_xsimd)
else()
    add_custom_target(xtest COMMAND test_xsimd DEPENDS test_xsimd)
endif()

if (XSIMD_ENABLE_WERROR)
    target_compile_options(test_xsimd PRIVATE -Werror -Wall -DXSIMD_SKIP_ON_WERROR)
endif()

add_subdirectory(doc)
add_subdirectory(architectures)

if(EMSCRIPTEN)
    set_target_properties(test_xsimd PROPERTIES LINK_FLAGS "-s MODULARIZE=1 -s EXPORT_NAME=test_xsimd_wasm -s WASM=1 -s ALLOW_MEMORY_GROWTH=1 -lembind")
    target_compile_options(test_xsimd
        PUBLIC --std=c++14
        PUBLIC "SHELL: -msimd128"
        PUBLIC "SHELL: -msse2"
    )
endif()
