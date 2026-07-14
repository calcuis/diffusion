cmake_minimum_required(VERSION 3.14)
project("diffusion" C CXX)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if (NOT XCODE AND NOT MSVC AND NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

if (MSVC)
    add_compile_definitions(_CRT_SECURE_NO_WARNINGS)
    add_compile_definitions(_SILENCE_CXX17_CODECVT_HEADER_DEPRECATION_WARNING)
    add_compile_options(
        $<$<COMPILE_LANGUAGE:C>:/MP>
        $<$<COMPILE_LANGUAGE:C>:/utf-8>
        $<$<COMPILE_LANGUAGE:CXX>:/MP>
        $<$<COMPILE_LANGUAGE:CXX>:/utf-8>
    )
endif()

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

if(APPLE)
    function(sd_set_macos_rpaths target)
        get_target_property(target_type ${target} TYPE)
        if(target_type STREQUAL "EXECUTABLE")
            set(runtime_paths "@executable_path" "@executable_path/../lib")
        elseif(target_type STREQUAL "SHARED_LIBRARY" OR target_type STREQUAL "MODULE_LIBRARY")
            set(runtime_paths "@loader_path" "@loader_path/../lib")
            set_target_properties(${target} PROPERTIES
                MACOSX_RPATH ON
                INSTALL_NAME_DIR "@rpath"
                BUILD_WITH_INSTALL_NAME_DIR ON
            )
        else()
            return()
        endif()
        set_target_properties(${target} PROPERTIES
            BUILD_RPATH "${runtime_paths}"
            INSTALL_RPATH "${runtime_paths}"
            BUILD_WITH_INSTALL_RPATH ON
        )
    endfunction()
endif()

#
# Options
#

option(SD_BUILD_CLI         "diffusion: build the diffusion executable"   ON)
option(SD_BUILD_SHARED_LIBS "diffusion: build shared library"             ON)
option(SD_CUDA              "diffusion: CUDA backend"                     OFF)
option(SD_HIPBLAS           "diffusion: ROCm/HIP backend"                 OFF)
option(SD_METAL             "diffusion: Metal backend"                    OFF)
option(SD_VULKAN            "diffusion: Vulkan backend"                   OFF)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED true)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED true)

if(SD_CUDA)
    message("-- Use CUDA as backend diffusion")
    set(GGML_CUDA ON)
endif()

if(SD_METAL)
    message("-- Use Metal as backend diffusion")
    set(GGML_METAL ON)
endif()

if(SD_VULKAN)
    message("-- Use Vulkan as backend diffusion")
    set(GGML_VULKAN ON)
endif()

if(SD_HIPBLAS)
    message("-- Use HIPBLAS as backend diffusion")
    set(GGML_HIP ON)
    # ggml-hip's device-stub objects must be position-independent, or the
    # default-PIE executable link fails on distros that default to PIE.
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
endif()

set(SD_LIB diffusion)

file(GLOB SD_LIB_SOURCES CONFIGURE_DEPENDS
    "src/*.h"
    "src/*.cpp"
    "src/*.hpp"
    "src/conditioning/*.h"
    "src/conditioning/*.cpp"
    "src/conditioning/*.hpp"
    "src/core/*.h"
    "src/core/*.cpp"
    "src/core/*.hpp"
    "src/extensions/*.h"
    "src/extensions/*.cpp"
    "src/extensions/*.hpp"
    "src/model/*/*.h"
    "src/model/*/*.cpp"
    "src/model/*/*.hpp"
    "src/runtime/*.h"
    "src/runtime/*.cpp"
    "src/runtime/*.hpp"
    "src/model_io/*.h"
    "src/model_io/*.cpp"
    "src/tokenizers/*.h"
    "src/tokenizers/*.cpp"
    "src/tokenizers/vocab/*.h"
    "src/tokenizers/vocab/*.cpp"
)

find_program(GIT_EXE NAMES git git.exe NO_CMAKE_FIND_ROOT_PATH)
if(GIT_EXE)
    execute_process(COMMAND ${GIT_EXE} describe --tags --abbrev=7 --dirty=+
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        OUTPUT_VARIABLE DIFFUSION_BUILD_VERSION
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
    )
    execute_process(COMMAND ${GIT_EXE} rev-parse --short HEAD
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        OUTPUT_VARIABLE DIFFUSION_BUILD_COMMIT
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
    )
endif()
if(NOT DIFFUSION_BUILD_VERSION)
    set(DIFFUSION_BUILD_VERSION unknown)
endif()
if(NOT DIFFUSION_BUILD_COMMIT)
    set(DIFFUSION_BUILD_COMMIT unknown)
endif()
message(STATUS "diffusion version ${DIFFUSION_BUILD_VERSION} (${DIFFUSION_BUILD_COMMIT})")

set_property(
  SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/src/version.cpp
  APPEND PROPERTY COMPILE_DEFINITIONS
  SDCPP_BUILD_COMMIT=${DIFFUSION_BUILD_COMMIT} SDCPP_BUILD_VERSION=${DIFFUSION_BUILD_VERSION}
)

if(SD_BUILD_SHARED_LIBS)
    message("-- Build shared library")
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
    add_library(${SD_LIB} SHARED ${SD_LIB_SOURCES})
    add_definitions(-DSD_BUILD_SHARED_LIB)
    target_compile_definitions(${SD_LIB} PRIVATE -DSD_BUILD_DLL)
else()
    message("-- Build static library")
    add_library(${SD_LIB} STATIC ${SD_LIB_SOURCES})
endif()

if(APPLE)
    sd_set_macos_rpaths(${SD_LIB})
endif()

set(CMAKE_POLICY_DEFAULT_CMP0077 NEW)

# see https://github.com/ggerganov/ggml/pull/682
add_definitions(-DGGML_MAX_NAME=128)

# vendored ggml kernels (static, linked into the diffusion library)
set(BUILD_SHARED_LIBS OFF)
add_subdirectory(kernels)

target_link_libraries(${SD_LIB} PUBLIC ggml)
target_include_directories(${SD_LIB} PUBLIC . src include)
target_include_directories(${SD_LIB} PRIVATE src/core)
target_include_directories(${SD_LIB} PUBLIC thirdparty)
target_compile_features(${SD_LIB} PUBLIC c_std_11 cxx_std_17)

if(SD_BUILD_CLI)
    add_subdirectory(cli)
endif()
