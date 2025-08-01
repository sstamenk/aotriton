# Copyright © 2023-2025 Advanced Micro Devices, Inc.
# SPDX-License-Identifier: MIT

cmake_minimum_required(VERSION 3.18 FATAL_ERROR)

project(AOTriton CXX C)

# Version numbers
set(AOTRITON_VERSION_MAJOR_INT 0)
set(AOTRITON_VERSION_MINOR_INT 11)
set(AOTRITON_VERSION_PATCH_INT 0)
# Must have integer-suffix, other 0 generates
# /* #undef AOTRITON_VERSION_MAJOR */
# in config.h
set(AOTRITON_VERSION_MAJOR "${AOTRITON_VERSION_MAJOR_INT}u")
set(AOTRITON_VERSION_MINOR "${AOTRITON_VERSION_MINOR_INT}u")
set(AOTRITON_VERSION_PATCH "${AOTRITON_VERSION_PATCH_INT}u")
execute_process(
  COMMAND git log -1 --format=%H
  WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
  OUTPUT_VARIABLE AOTRITON_GIT_SHA1
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_C_USE_RESPONSE_FILE_FOR_OBJECTS   ON)
set(CMAKE_CXX_USE_RESPONSE_FILE_FOR_OBJECTS ON)
set(CMAKE_NINJA_FORCE_RESPONSE_FILE         ON)
set(CMAKE_CXX_USE_RESPONSE_FILE_FOR_LIBRARIES ON)
find_program(CCACHE_FOUND ccache)
if(CCACHE_FOUND)
  set(CMAKE_C_COMPILER_LAUNCHER ${CCACHE_FOUND})
  set(CMAKE_CXX_COMPILER_LAUNCHER ${CCACHE_FOUND})
  message(STATUS "[AOTriton] ccache found and enabled")
else()
  message(STATUS "[AOTriton] ccache not found")
endif()
message(STATUS "[AOTriton] Triton's caching system is parallel to ccache and always enabled.")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -finput-charset=UTF-8")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -finput-charset=UTF-8")
message(STATUS "[AOTriton] UTF-8 input character is set for all C/C++ source files.")

set(AOTRITON_MIN_PYTHON 3.10 CACHE STRING "Minimal Python version for find_package.")
if(AOTRITON_MIN_PYTHON VERSION_LESS "3.10")
  message(FATAL_ERROR "Do not set AOTRITON_MIN_PYTHON lower than 3.10. The code itself does not support it.")
endif()

find_package(Python3 ${AOTRITON_MIN_PYTHON} COMPONENTS Interpreter REQUIRED)
# Must set this otherwise hip-config-amd.cmake cannot find AMDDeviceLibs
list(APPEND CMAKE_PREFIX_PATH "/opt/rocm")
find_package(hip REQUIRED)

set(VENV_DIR "${CMAKE_CURRENT_BINARY_DIR}/venv" CACHE STRING "Virtual Environment Directory")
option(AOTRITON_NO_PYTHON "Disable python binding build" OFF)
option(AOTRITON_ENABLE_ASAN "Enable Address Sanitizer. Implies -g" OFF)
option(AOTRITON_BUILD_FOR_TUNING "Build all GPU kernels and set -DAOTRITON_BUILD_FOR_TUNING=1 (=0 otherwise)" OFF)
set(AOTRITON_BUILD_FOR_TUNING_BUT_SKIP_KERNEL "" CACHE STRING "Use tuning database for certain kernels when AOTRITON_BUILD_FOR_TUNING=ON")
option(AOTRITON_ENABLE_FP32_INPUTS "Enable FP32 support." ON)
option(AOTRITON_NOIMAGE_MODE "Only build C++ Shim part. Kernel image builds are disabled" OFF)
set(AOTRITON_GPU_BUILD_TIMEOUT "8.0" CACHE STRING "GPU kernel compiler times out after X minutes. 0 for indefinite. Highly recommended if AOTRITON_BUILD_FOR_TUNING=On.")
set(AOTRITON_TARGET_ARCH "gfx90a;gfx942;gfx950;gfx1100;gfx1151;gfx1150;gfx1201;gfx1200" CACHE STRING "Target GPU Architecture. Select all GPUs within the given list")
set(TARGET_GPUS "OBSOLETE" CACHE STRING "OBSOLETE. To select only one GPU, use AOTRITON_TARGET_ARCH or AOTRITON_OVERRIDE_TARGET_GPUS.")
set(AOTRITON_OVERRIDE_TARGET_GPUS "gfx1101_mod0" CACHE STRING "Override AOTRITON_TARGET_ARCH, and only build for GPUs within this list.")

# Early failure for python binding
if(NOT AOTRITON_NO_PYTHON)
  find_package(Python3 ${AOTRITON_MIN_PYTHON} COMPONENTS Development REQUIRED)
  add_subdirectory(third_party/pybind11)
  if(NOT EXISTS ${CMAKE_SOURCE_DIR}/third_party/pybind11/CMakeLists.txt)
    message(FATAL_ERROR "third_party/pybind11 is not initialized\nRun `git submodule sync && git submodule update --init --recursive --force` to fix it.")
  endif()
endif()

# Resolve name conflicts with suffix
set(AOTRITON_NAME_SUFFIX "" CACHE STRING "Add suffix to namespace and library file name. This is to resolve name conflicts with PyTorch's AOTriton during testing.")
if(AOTRITON_NAME_SUFFIX)
  set(AOTRITON_ENABLE_SUFFIX ON)
else()
  set(AOTRITON_ENABLE_SUFFIX OFF)
endif()

if(NOT TARGET_GPUS STREQUAL "OBSOLETE")
  message(FATAL_ERROR "TARGET_GPUS is OBSOLETE in Dispatcher V3. Use AOTRITON_TARGET_ARCH or AOTRITON_OVERRIDE_TARGET_GPUS.")
endif()

include(GNUInstallDirs)
configure_file(include/aotriton/config.h.in include/aotriton/config.h)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/include/aotriton/config.h
  DESTINATION ${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_INCLUDEDIR}/aotriton)

# Kernel Storage V2 uses xz/LZMA for compression
include(FindPkgConfig)
pkg_search_module(LZMA REQUIRED liblzma)
add_library(lzma_interface INTERFACE)
target_link_libraries(lzma_interface INTERFACE ${LZMA_LIBRARIES})
target_link_directories(lzma_interface INTERFACE ${LZMA_LIBRARY_DIRS})
target_include_directories(lzma_interface INTERFACE ${LZMA_INCLUDE_DIRS})

if(AOTRITON_ENABLE_ASAN)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -fsanitize=address -fno-omit-frame-pointer")
endif()

set(Python_ARTIFACTS_INTERACTIVE TRUE)

# Not a target, we need to override Python3_EXECUTABLE later
execute_process(COMMAND "${Python3_EXECUTABLE}" -m venv "${VENV_DIR}")

set(ENV{VIRTUAL_ENV} "${VENV_DIR}")
message("VENV_DIR ${VENV_DIR}")
# set(Python3_FIND_VIRTUALENV FIRST)
# unset(Python3_EXECUTABLE)
# find_package(Python3 COMPONENTS Interpreter REQUIRED)

execute_process(COMMAND ${CMAKE_COMMAND} -E env VIRTUAL_ENV=${VENV_DIR} "${VENV_DIR}/bin/python" -c "import site; print(site.getsitepackages()[0], end='')" OUTPUT_VARIABLE VENV_SITE)
# string(STRIP "${VENV_SITE}" VENV_SITE)
message("VENV_SITE ${VENV_SITE}")

execute_process(COMMAND ${CMAKE_COMMAND} -E env VIRTUAL_ENV=${VENV_DIR} "${VENV_DIR}/bin/python" -m pip install -r "${CMAKE_CURRENT_LIST_DIR}/requirements.txt")

# AOTRITON_NOIMAGE_MODE does not need Triton
if(NOT AOTRITON_NOIMAGE_MODE)
  set(TRITON_BUILD_DIR "${CMAKE_CURRENT_BINARY_DIR}/triton_build")
  execute_process(COMMAND ${CMAKE_COMMAND} -E make_directory "${TRITON_BUILD_DIR}")
  # set(AOTRITON_TRITON_EGGLINK "${VENV_SITE}/triton.egg-link")
  # message("AOTRITON_TRITON_EGGLINK ${AOTRITON_TRITON_EGGLINK}")
  set(AOTRITON_TRITON_SO "${VENV_SITE}/triton/_C/libtriton.so")

  add_custom_command(OUTPUT "${AOTRITON_TRITON_SO}"
    COMMAND ${CMAKE_COMMAND} -E env TRITON_BUILD_PROTON=OFF VIRTUAL_ENV=${VENV_DIR} TRITON_BUILD_DIR=${TRITON_BUILD_DIR} "${VENV_DIR}/bin/python" -m pip install .
    # COMMAND ${CMAKE_COMMAND} -E env VIRTUAL_ENV=${VENV_DIR} python -m pip show triton
    WORKING_DIRECTORY "${CMAKE_CURRENT_LIST_DIR}/third_party/triton/"
  )
  add_custom_target(aotriton_venv_triton ALL DEPENDS "${AOTRITON_TRITON_SO}")
endif(NOT AOTRITON_NOIMAGE_MODE)

add_subdirectory(v3src)

if(NOT AOTRITON_NO_PYTHON)
  add_subdirectory(bindings) # FIXME: compile python binding
endif()
