cmake_minimum_required(VERSION 3.12)
project(FREETYPE_CMAKE)

set(CMAKE_POLICY_DEFAULT_CMP0114 NEW) # CMake is generating for the Xcode "new build system"

include(ExternalProject)

# adapted from https://github.com/roxlu/cmake-freetype-harfbuzz/blob/master/CMakeLists.txt

# freetype requires: bzip2, libpng, brotlidec, zlib, hurfbuzz
# harfbuz requires freetype

# add zlib

set(SYSROOT ${CMAKE_CURRENT_BINARY_DIR}/EXTERNAL_PROJECT_INSTALLED_FILES)

ExternalProject_Add(
    external_project_zlib
        SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/zlib
    CMAKE_ARGS
        "-DCMAKE_INSTALL_PREFIX=${SYSROOT}"
        -DCMAKE_POLICY_DEFAULT_CMP0042:STRING=NEW
)

# add libpng, depends on zlib

ExternalProject_Add(
    external_project_png
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/libpng
    CMAKE_ARGS
        -DPNG_SHARED=FALSE
        -DPNG_STATIC=TRUE
        -DPNG_HARDWARE_OPTIMIZATIONS=TRUE
        -DPNG_EXECUTABLES=FALSE
        -DPNG_TESTS=FALSE
        "-DCMAKE_INSTALL_PREFIX=${SYSROOT}"
        -DCMAKE_POLICY_DEFAULT_CMP0074:STRING=NEW
        -DCMAKE_POLICY_DEFAULT_CMP0114:STRING=NEW
        "-DZLIB_ROOT=${SYSROOT}"
    DEPENDS external_project_zlib
)

# add bzip2

ExternalProject_Add(
    external_project_bzip2
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/bzip2
    CMAKE_ARGS
        "-DCMAKE_INSTALL_PREFIX=${SYSROOT}"
)

# add brotli

ExternalProject_Add(
    external_project_brotli
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/brotli
    CMAKE_ARGS
        "-DCMAKE_INSTALL_PREFIX=${SYSROOT}"
        -DCMAKE_POLICY_DEFAULT_CMP0042:STRING=NEW
)

ExternalProject_Add(
    external_project_freetype_no_harfbuzz
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/freetype-2.10.4
    CMAKE_ARGS
        -DFT_WITH_PNG=TRUE
        -DFT_WITH_ZLIB=TRUE
        -DFT_WITH_BZIP2=TRUE
        -DFT_WITH_HARFBUZZ=FALSE
        -DCMAKE_DISABLE_FIND_PACKAGE_harfbuzz=TRUE
        "-DCMAKE_INSTALL_PREFIX=${SYSROOT}"
        -DCMAKE_POLICY_DEFAULT_CMP0074:STRING=NEW
        -DCMAKE_POLICY_DEFAULT_CMP0114:STRING=NEW
        -DCMAKE_FIND_USE_CMAKE_SYSTEM_PATH=FALSE
        -DCMAKE_FIND_USE_SYSTEM_ENVIRONMENT_PATH=FALSE
        "-DZLIB_ROOT=${SYSROOT}"
        "-DPNG_ROOT=${SYSROOT}"
        "-DBZip2_ROOT=${SYSROOT}"
        "-Dbrotlidec_ROOT=${SYSROOT}"
    DEPENDS
        external_project_png
        external_project_bzip2
        external_project_brotli
)

# harfbuzz uses the cmake module findFreetype
# harfbuzz supports GRAPHITE2
# harfbuzz supports ICU

ExternalProject_Add(
    external_project_graphite
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/graphite
    CMAKE_ARGS
        -DBUILD_SHARED_LIBS=FALSE
        -DGRAPHITE2_NFILEFACE=TRUE
        "-DCMAKE_INSTALL_PREFIX=${SYSROOT}"
)

# test for the existence of ICU library

set(HAVE_ICU FALSE)

find_package(PkgConfig)
pkg_check_modules(PC_ICU QUIET icu-uc)

find_path(ICU_INCLUDE_DIR NAMES unicode/utypes.h HINTS ${PC_ICU_INCLUDE_DIRS} ${PC_ICU_INCLUDEDIR})
find_library(ICU_LIBRARY NAMES libicuuc cygicuuc cygicuuc32 icuuc HINTS ${PC_ICU_LIBRARY_DIRS} ${PC_ICU_LIBDIR})

message(ICU_INCLUDE_DIR = ${ICU_INCLUDE_DIR})
message(ICU_LIBRARY = ${ICU_LIBRARY})

if (NOT ICU_LIBRARY)
    include(FindICU)
    message(ICU_FOUND = ${ICU_FOUND})
    message(ICU_INCLUDE_DIR = ${ICU_INCLUDE_DIR})
    message(ICU_LIBRARY = ${ICU_LIBRARY})
    if (NOT ICU_FOUND)
    else()
        set(HAVE_ICU TRUE)
    endif()
else()
    set(HAVE_ICU TRUE)
endif()

message(HAVE_ICU = ${HAVE_ICU})

if (APPLE)
    set(HAVE_CORE_TEXT TRUE)
    set(FREETYPE_REQUIRED_APPLE_FRAMEWORKS "-framework CoreFoundation -framework CoreGraphics -framework CoreText")
else()
    set(HAVE_CORE_TEXT FALSE)
    set(FREETYPE_REQUIRED_APPLE_FRAMEWORKS "")
endif()

ExternalProject_Add(
    external_project_harfbuzz
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/harfbuzz
    CMAKE_ARGS
        -DHB_BUILD_SUBSET=TRUE
        -DHB_HAVE_FREETYPE=TRUE
        -DHB_HAVE_GRAPHITE2=TRUE
        -DHB_HAVE_ICU=${HAVE_ICU}
        "-DICU_INCLUDE_DIR=${ICU_INCLUDE_DIR}"
        "-DICU_LIBRARY=${ICU_LIBRARY}"
        -DHB_HAVE_CORETEXT=${HAVE_CORE_TEXT}
        "-DCMAKE_INSTALL_PREFIX=${SYSROOT}"
        "-DCMAKE_PREFIX_PATH=${SYSROOT}"
        -DCMAKE_POLICY_DEFAULT_CMP0074:STRING=NEW
        -DCMAKE_POLICY_DEFAULT_CMP0114:STRING=NEW
        "-DFREETYPE_DIR=${SYSROOT}"
        "-DGRAPHITE2_ROOT=${SYSROOT}"
    DEPENDS
        external_project_freetype_no_harfbuzz
        external_project_graphite
)

ExternalProject_Add(
    external_project_freetype
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/freetype-2.10.4
    CMAKE_ARGS
        -DFT_WITH_PNG=TRUE
        -DFT_WITH_ZLIB=TRUE
        -DFT_WITH_BZIP2=TRUE
        -DFT_WITH_HARFBUZZ=TRUE
        "-DCMAKE_INSTALL_PREFIX=${SYSROOT}"
        -DCMAKE_POLICY_DEFAULT_CMP0074:STRING=NEW
        -DCMAKE_POLICY_DEFAULT_CMP0114:STRING=NEW
        -DCMAKE_FIND_USE_CMAKE_SYSTEM_PATH=FALSE
        -DCMAKE_FIND_USE_SYSTEM_ENVIRONMENT_PATH=FALSE
        "-DZLIB_ROOT=${SYSROOT}"
        "-DPNG_ROOT=${SYSROOT}"
        "-DBZip2_ROOT=${SYSROOT}"
        "-Dbrotlidec_ROOT=${SYSROOT}"
        "-Dharfbuzz_ROOT=${SYSROOT}"
    DEPENDS
        external_project_harfbuzz
)


# link libraries to external libs

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR
        CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    if(PLATFORM_APPLE)
        set(WHOLE_ARCHIVE_FLAG "-Wl,-all_load" CACHE INTERNAL "all_load flag")
        set(NO_WHOLE_ARCHIVE_FLAG "-Wl,-noall_load" CACHE INTERNAL "noall_load flag")
    else()
        set(WHOLE_ARCHIVE_FLAG "-Wl,--whole-archive" CACHE INTERNAL "whole-archive flag")
        set(NO_WHOLE_ARCHIVE_FLAG "-Wl,--no-whole-archive" CACHE INTERNAL "no-whole-archive flag")
    endif()
else()
    set(WHOLE_ARCHIVE_FLAG "")
    set(NO_WHOLE_ARCHIVE_FLAG "")
endif()

add_library(zlib SHARED empty.cpp)
add_library(png SHARED empty.cpp)
add_library(bzip2 SHARED empty.cpp)
add_library(brotli SHARED empty.cpp)
add_library(graphite SHARED empty.cpp)
add_library(harfbuzz SHARED empty.cpp)
add_library(freetype SHARED empty.cpp)

add_dependencies(zlib external_project_zlib)
add_dependencies(png zlib external_project_png)
add_dependencies(bzip2 external_project_bzip2)
add_dependencies(brotli external_project_brotli)
add_dependencies(graphite external_project_graphite)
add_dependencies(harfbuzz png bzip2 brotli graphite external_project_freetype)
add_dependencies(freetype harfbuzz)

target_include_directories(zlib PUBLIC ${SYSROOT}/include)
target_include_directories(png PUBLIC ${SYSROOT}/include)
target_include_directories(bzip2 PUBLIC ${SYSROOT}/include)
target_include_directories(brotli PUBLIC ${SYSROOT}/include)
target_include_directories(graphite PUBLIC ${SYSROOT}/include)
target_include_directories(harfbuzz PUBLIC ${SYSROOT}/include)
target_include_directories(freetype PUBLIC ${SYSROOT}/include/freetype2)

if (APPLE)
    set(SH dylib)
    set(ST a)
    set(R lib)
elseif(UNIX)
    set(SH so)
    set(ST a)
    set(R lib)
elseif(WIN32)
    set(SH dll)
    set(ST lib)
    set(R lib)
endif()

# libraries:
# libbrotlicommon-static.a
# libbrotlidec-static.a
# libbrotlienc-static.a
# libbz2d.a
# libfreetyped.a
# libgraphite2.a
# libharfbuzz.a
# libpng.a -> libpng16d.a
# libpng16d.a
# libz.a

target_link_libraries(
        zlib
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libz.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
)

target_link_libraries(
        png
        zlib

        # libpng is a symlink to libpng16d
        # including both libpng and libpng16d results in duplicate symbols
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libpng.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
)

target_link_libraries(
        bzip2
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libbz2d.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
)
target_link_libraries(
        brotli
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libbrotlicommon-static.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libbrotlienc-static.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libbrotlidec-static.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
)
target_link_libraries(
        graphite
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libgraphite2.${ST} ${WHOLE_ARCHIVE_FLAG}
)

target_link_libraries(
        harfbuzz
        png
        bzip2
        brotli
        graphite
        ${FREETYPE_REQUIRED_APPLE_FRAMEWORKS}
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libharfbuzz.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libfreetyped.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
)
target_link_libraries(
        freetype
        harfbuzz
)