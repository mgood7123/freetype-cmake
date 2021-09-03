cmake_minimum_required(VERSION 3.10)
project(FREETYPE_CMAKE)

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

if (ANDROID)
    # for some reason ExternalProject does not work on android
    # so i cannot link freetype to harfbuzz and harfbuzz to freetype
    # i do not know how to link harfbuzz to freetype on android
    # harfbuzz cannot be found on android
    # graphite test fails on android
    # ninja: error: 'OUTPUT/libbz2.so', needed by 'OUTPUT/libfreetyped.so', missing and no known rule to make it

    # use old cmake that is known to work

    macro(freetype_cmake_set_library_prefix libpng bzip2 brotli harfbuzz freetype)
        set(PNG_LIBRARY ${libpng}/libpng.${FREETYPE_CMAKE_LIB_LIBRARY_SUFFIX} ${libpng}/libpng16d.${FREETYPE_CMAKE_LIB_LIBRARY_SUFFIX})
        set(PNG_LIBRARIES ${libpng}/libpng.${FREETYPE_CMAKE_LIB_LIBRARY_SUFFIX} ${libpng}/libpng16d.${FREETYPE_CMAKE_LIB_LIBRARY_SUFFIX})
        set(BZIP2_LIBRARIES ${bzip2}/libbz2.${FREETYPE_CMAKE_LIB_LIBRARY_SUFFIX})
        set(BROTLIDEC_LIBRARIES ${brotli}/libbrotlidec.${FREETYPE_CMAKE_LIB_LIBRARY_SUFFIX} ${brotli}/libbrotlienc.${FREETYPE_CMAKE_LIB_LIBRARY_SUFFIX} ${brotli}/libbrotlicommon.${FREETYPE_CMAKE_LIB_LIBRARY_SUFFIX})
        set(HARFBUZZ_LIBRARIES ${harfbuzz}/libharfbuzz.a)
        set(FREETYPE_LIBRARY ${freetype}/libfreetyped.a)
    endmacro()

    macro(freetype_cmake_set_library_prefixAll location)
        freetype_cmake_set_library_prefix(${location} ${location} ${location} ${location} ${location})
    endmacro()

    macro(freetype_cmake_include_freetype path_to_self)
        # quiet some warnings
        set(CMAKE_POLICY_DEFAULT_CMP0042 NEW) # ``MACOSX_RPATH`` is enabled by default.
        set(CMAKE_POLICY_DEFAULT_CMP0075 NEW) # Include file check macros honor ``CMAKE_REQUIRED_LIBRARIES``.

        # force dependant libs to compile so we have library paths to link to

        # freetype requires: bzip2, libpng, brotlidec, hurfbuzz

        set(BUILD_SHARED_LIBS OFF CACHE BOOL "" FORCE)

        set(PNG_SHARED ON CACHE BOOL "" FORCE)
        set(PNG_STATIC OFF CACHE BOOL "" FORCE)
        set(PNG_EXECUTABLES OFF CACHE BOOL "" FORCE)
        set(PNG_TESTS OFF CACHE BOOL "" FORCE)
        set(SKIP_INSTALL_ALL OFF CACHE BOOL "" FORCE)

        # subset and harfbuff results in duplicate symbols
        set(HB_BUILD_SUBSET OFF CACHE BOOL "" FORCE)
        set(HB_HAVE_FREETYPE ON CACHE BOOL "" FORCE)
        # HB_HAVE_FREETYPE requires cmake standard 11
        if (NOT CMAKE_CXX_STANDARD OR CMAKE_CXX_STANDARD LESS 11)
            set(CMAKE_CXX_STANDARD 11)
            set(CMAKE_CXX_STANDARD_REQUIRED ON)
        endif()

        set(BROTLI_DISABLE_TESTS ON CACHE BOOL "" FORCE)

        set(FT_WITH_BZIP2 ON CACHE BOOL "" FORCE)
        set(FT_WITH_PNG ON CACHE BOOL "" FORCE)
        set(FT_WITH_BROTLI ON CACHE BOOL "" FORCE)
        set(FT_WITH_HARFBUZZ ON CACHE BOOL "" FORCE)

        if (APPLE)
            set(FREETYPE_CMAKE_LIB_LIBRARY_SUFFIX dylib)
        elseif(UNIX)
            set(FREETYPE_CMAKE_LIB_LIBRARY_SUFFIX so)
        elseif(WIN32)
            set(FREETYPE_CMAKE_LIB_LIBRARY_SUFFIX lib)
        endif()

        # set library defs
        if (CMAKE_LIBRARY_OUTPUT_DIRECTORY)
            set(FREETYPE_PREFEX_LIB_OUT_DIR ${CMAKE_LIBRARY_OUTPUT_DIRECTORY})
            freetype_cmake_set_library_prefixAll(${FREETYPE_PREFEX_LIB_OUT_DIR})
        else()
            set(FREETYPE_PREFEX_LIB_OUT_DIR ${CMAKE_CURRENT_BINARY_DIR})
            freetype_cmake_set_library_prefix(
                    ${FREETYPE_PREFEX_LIB_OUT_DIR}/libpng
                    ${FREETYPE_PREFEX_LIB_OUT_DIR}/bzip2
                    ${FREETYPE_PREFEX_LIB_OUT_DIR}/brotli
                    ${FREETYPE_PREFEX_LIB_OUT_DIR}/harfbuzz
                    ${FREETYPE_PREFEX_LIB_OUT_DIR}/freetype-2.10.4
            )
        endif()

        # set include defs
        set(PNG_PNG_INCLUDE_DIR ${path_to_self}/libpng)
        set(BZIP2_INCLUDE_DIR ${path_to_self}/bzip2)
        set(BROTLIDEC_INCLUDE_DIRS ${path_to_self}/brotli/c/include)
        set(HARFBUZZ_INCLUDE_DIRS ${path_to_self}/harfbuzz/src)
        set(FREETYPE_INCLUDE_DIRS ${path_to_self}/freetype-2.10.4/include)

        # include files
        include_directories(${PNG_PNG_INCLUDE_DIR})
        # generated files from libpng during build
        include_directories(${CMAKE_CURRENT_BINARY_DIR}/libpng)
        include_directories(${BZIP2_INCLUDE_DIR})
        include_directories(${BROTLIDEC_INCLUDE_DIRS})
        include_directories(${HARFBUZZ_INCLUDE_DIRS})
        include_directories(${FREETYPE_INCLUDE_DIRS})

        # prepare projects
        add_subdirectory(${path_to_self}/libpng ${CMAKE_CURRENT_BINARY_DIR}/libpng)
        add_subdirectory(${path_to_self}/bzip2 ${CMAKE_CURRENT_BINARY_DIR}/bzip2)
        add_subdirectory(${path_to_self}/brotli ${CMAKE_CURRENT_BINARY_DIR}/brotli)
        add_subdirectory(${path_to_self}/harfbuzz ${CMAKE_CURRENT_BINARY_DIR}/harfbuzz)
        add_subdirectory(${path_to_self}/freetype-2.10.4 ${CMAKE_CURRENT_BINARY_DIR}/freetype-2.10.4)

        # build projects
        add_library(FREETYPE_CMAKE_LIB_FREETYPE SHARED ${path_to_self}/empty.cpp)

        add_dependencies(libz2 png)
        add_dependencies(brotlicommon libz2)
        add_dependencies(brotlienc brotlicommon)
        add_dependencies(brotlidec brotlienc)
        add_dependencies(harfbuzz brotlidec)
        add_dependencies(freetype harfbuzz)
    endmacro()

    freetype_cmake_include_freetype(${CMAKE_CURRENT_SOURCE_DIR})

    add_subdirectory(zlib)
    add_library(zlib-cmake SHARED empty.cpp)
    add_dependencies(zlib-cmake zlib)
    target_include_directories(zlib-cmake PUBLIC zlib)
    target_link_libraries(zlib-cmake zlib)

    add_library(png-cmake SHARED empty.cpp)
    add_library(bzip2-cmake SHARED empty.cpp)
    add_library(brotli-cmake SHARED empty.cpp)
    add_library(harfbuzz-cmake SHARED empty.cpp)
    add_library(freetype-cmake SHARED empty.cpp)

    add_dependencies(png-cmake png)
    add_dependencies(bzip2-cmake libz2)
    add_dependencies(brotli-cmake brotlidec brotlienc brotlicommon)
    add_dependencies(harfbuzz-cmake png-cmake bzip2-cmake brotli-cmake harfbuzz freetype)
    add_dependencies(freetype-cmake harfbuzz-cmake)

    # include files
    target_include_directories(png-cmake PUBLIC ${PNG_PNG_INCLUDE_DIR} ${CMAKE_CURRENT_BINARY_DIR}/libpng)
    target_include_directories(bzip2-cmake PUBLIC ${BZIP2_INCLUDE_DIR})
    target_include_directories(brotli-cmake PUBLIC ${BROTLIDEC_INCLUDE_DIRS})
    target_include_directories(harfbuzz-cmake PUBLIC ${HARFBUZZ_INCLUDE_DIRS})
    target_include_directories(freetype-cmake PUBLIC ${FREETYPE_INCLUDE_DIRS})

    target_link_libraries(
            png-cmake
            zlib-cmake
            png
    )

    target_link_libraries(
            bzip2-cmake
            libz2
    )
    target_link_libraries(
            brotli-cmake
            brotlicommon
            brotlienc
            brotlidec
    )
    target_link_libraries(
            harfbuzz-cmake
            png-cmake
            bzip2-cmake
            brotli-cmake
            ${WHOLE_ARCHIVE_FLAG} ${HARFBUZZ_LIBRARIES} ${NO_WHOLE_ARCHIVE_FLAG}
            ${WHOLE_ARCHIVE_FLAG} ${FREETYPE_LIBRARY} ${NO_WHOLE_ARCHIVE_FLAG}
    )
    target_link_libraries(
            freetype-cmake
            harfbuzz-cmake
    )
else()
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

# the name of png, resulting in libpng.so
# conflicts with the system library libPng.so
# in /System/Library/Frameworks/ImageIO.framework/Resources/libPng.dylib
# so add -cmake suffix to every library

add_library(zlib-cmake SHARED empty.cpp)
add_library(png-cmake SHARED empty.cpp)
add_library(bzip2-cmake SHARED empty.cpp)
add_library(brotli-cmake SHARED empty.cpp)
add_library(graphite-cmake SHARED empty.cpp)
add_library(harfbuzz-cmake SHARED empty.cpp)
add_library(freetype-cmake SHARED empty.cpp)

add_dependencies(zlib-cmake external_project_zlib)
add_dependencies(png-cmake zlib-cmake external_project_png)
add_dependencies(bzip2-cmake external_project_bzip2)
add_dependencies(brotli-cmake external_project_brotli)
add_dependencies(graphite-cmake external_project_graphite)
add_dependencies(harfbuzz-cmake png-cmake bzip2-cmake brotli-cmake graphite-cmake external_project_freetype)
add_dependencies(freetype-cmake harfbuzz-cmake)

target_include_directories(zlib-cmake PUBLIC ${SYSROOT}/include)
target_include_directories(png-cmake PUBLIC ${SYSROOT}/include)
target_include_directories(bzip2-cmake PUBLIC ${SYSROOT}/include)
target_include_directories(brotli-cmake PUBLIC ${SYSROOT}/include)
target_include_directories(graphite-cmake PUBLIC ${SYSROOT}/include)
target_include_directories(harfbuzz-cmake PUBLIC ${SYSROOT}/include)
target_include_directories(freetype-cmake PUBLIC ${SYSROOT}/include/freetype2)

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
        zlib-cmake
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libz.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
)

target_link_libraries(
        png-cmake
        zlib-cmake

        # libpng is a symlink to libpng16d
        # including both libpng and libpng16d results in duplicate symbols
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libpng.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
)

target_link_libraries(
        bzip2-cmake
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libbz2d.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
)
target_link_libraries(
        brotli-cmake
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libbrotlicommon-static.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libbrotlienc-static.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libbrotlidec-static.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
)
target_link_libraries(
        graphite-cmake
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libgraphite2.${ST} ${WHOLE_ARCHIVE_FLAG}
)

target_link_libraries(
        harfbuzz-cmake
        png-cmake
        bzip2-cmake
        brotli-cmake
        graphite-cmake
        ${FREETYPE_REQUIRED_APPLE_FRAMEWORKS}
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libharfbuzz.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
        ${WHOLE_ARCHIVE_FLAG} ${SYSROOT}/${R}/libfreetyped.${ST} ${NO_WHOLE_ARCHIVE_FLAG}
)
target_link_libraries(
        freetype-cmake
        harfbuzz-cmake
)
endif()