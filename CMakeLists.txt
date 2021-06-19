cmake_minimum_required(VERSION 3.4)
project(FREEMAKE_CMAKE)

# quiet some warnings
set(CMAKE_POLICY_DEFAULT_CMP0042 NEW) # ``MACOSX_RPATH`` is enabled by default.
set(CMAKE_POLICY_DEFAULT_CMP0075 NEW) # Include file check macros honor ``CMAKE_REQUIRED_LIBRARIES``.

# force dependant libs to compile so we have library paths to link to

# freetype requires: libpng, brotlidec, hurfbuzz

set(FT_WITH_BZIP2 ON CACHE BOOL "" FORCE)
set(FT_WITH_PNG ON CACHE BOOL "" FORCE)
set(FT_WITH_BROTLI ON CACHE BOOL "" FORCE)
set(FT_WITH_HARFBUZZ ON CACHE BOOL "" FORCE)

# test builder makes everything easy cus fk remembering cmake
if(NOT COMMAND testBuilder_build)
    add_subdirectory(testBuilder)
    testBuilder_set_current_working_directory_to_default_binary_directory()
endif()

# setup sources
testBuilder_add_source(FREEMAKE_CMAKE_LIB_BZIP2 empty.cpp)
testBuilder_add_source(FREEMAKE_CMAKE_LIB_LIBPNG empty.cpp)
testBuilder_add_source(FREEMAKE_CMAKE_LIB_BROTLIDEC empty.cpp)
testBuilder_add_source(FREEMAKE_CMAKE_LIB_HARFBUZZ empty.cpp)
testBuilder_add_source(FREEMAKE_CMAKE_LIB_FREETYPE empty.cpp)

# setup libs
testBuilder_add_library(FREEMAKE_CMAKE_LIB_LIBZIP2 bzip2)
testBuilder_add_library(FREEMAKE_CMAKE_LIB_LIBPNG libpng)
testBuilder_add_library(FREEMAKE_CMAKE_LIB_BROTLIDEC brotlidec)
testBuilder_add_library(FREEMAKE_CMAKE_LIB_HARFBUZZ harfbuzz)
testBuilder_add_library(FREEMAKE_CMAKE_LIB_FREETYPE freetype)

# build libpng

# pnglibconf.h might not exist depending on the version used
if(NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/libpng/pnglibconf.h)
    file(MAKE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/TMP___)

    file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/libpng/scripts/pnglibconf.h.prebuilt DESTINATION ${CMAKE_CURRENT_SOURCE_DIR}/TMP___)

    if(NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/TMP___/pnglibconf.h.prebuilt)
        file(REMOVE_RECURSIVE ${CMAKE_CURRENT_SOURCE_DIR}/TMP___)
        message(FATAL_ERROR "failed to copy pnglibconf.h.prebuilt")
    endif()

    file(RENAME ${CMAKE_CURRENT_SOURCE_DIR}/TMP___/pnglibconf.h.prebuilt ${CMAKE_CURRENT_SOURCE_DIR}/libpng/pnglibconf.h)

    if(NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/libpng/pnglibconf.h)
        file(REMOVE_RECURSIVE ${CMAKE_CURRENT_SOURCE_DIR}/TMP___)
        message(FATAL_ERROR "failed to move pnglibconf.h.prebuilt")
    endif()

    file(REMOVE_RECURSIVE ${CMAKE_CURRENT_SOURCE_DIR}/TMP___)
endif()

add_subdirectory(libpng)
testBuilder_build_static_library(FREEMAKE_CMAKE_LIB_LIBPNG)

#./debug_BUILD/libpng/libpng.a
#./debug_BUILD/libpng/libpng16d.a
set(PNG_LIBRARY ${CMAKE_CURRENT_BINARY_DIR}/libpng/libpng.a)

set(PNG_PNG_INCLUDE_DIR libpng)
include_directories(${PNG_PNG_INCLUDE_DIR})

# build bzip2
add_subdirectory(bzip2)
testBuilder_build_static_library(FREEMAKE_CMAKE_LIB_BZIP2)

#./debug_BUILD/bzip2/libbz2d.a
set(BZIP2_LIBRARY ${CMAKE_CURRENT_BINARY_DIR}/bzip2/libbz2d.a)

set(BZIP2_INCLUDE_DIR bzip2)
include_directories(${BZIP2_INCLUDE_DIR})

# build brotlidec
add_subdirectory(brotli)
testBuilder_build_static_library(FREEMAKE_CMAKE_LIB_BROTLIDEC)

#./debug_BUILD/brotli/libbrotlidec-static.a
#./debug_BUILD/brotli/libbrotlienc-static.a
#./debug_BUILD/brotli/libbrotlicommon-static.a
set(BROTLIDEC_LIBRARIES ${CMAKE_CURRENT_BINARY_DIR}/brotli/libbrotlidec-static.a)

set(BROTLIDEC_INCLUDE_DIRS brotli/c/include)
include_directories(${BROTLIDEC_INCLUDE_DIRS})

# build harfbuzz
add_subdirectory(harfbuzz)
testBuilder_build_static_library(FREEMAKE_CMAKE_LIB_HARFBUZZ)

#./debug_BUILD/harfbuzz/libharfbuzz.a
#./debug_BUILD/harfbuzz/libharfbuzz-subset.a
set(HARFBUZZ_LIBRARIES ${CMAKE_CURRENT_BINARY_DIR}/harfbuzz/libharfbuzz.a)

set(HARFBUZZ_INCLUDE_DIRS harfbuzz/src)
include_directories(${HARFBUZZ_INCLUDE_DIRS})

# build freetype
add_subdirectory(freetype-2.10.4)
testBuilder_build_static_library(FREEMAKE_CMAKE_LIB_FREETYPE)

#./debug_BUILD/freetype-2.10.4/libfreetyped.a
set(FREETYPE_LIBRARY ${CMAKE_CURRENT_BINARY_DIR}/Debug/lib/libfreetyped.a)

set(FREETYPE_INCLUDE_DIRS freetype-2.10.4/include)
include_directories(${FREETYPE_INCLUDE_DIRS})

testBuilder_add_source(test empty.cpp)
testBuilder_add_library(test freetype)
testBuilder_build_static_library(test)