cmake_minimum_required(VERSION 3.4)
project(FREETYPE_CMAKE)

if (APPLE)
    set(FREETYPE_LIBRARY_SUFFIX dylib)
elseif(UNIX)
    set(FREETYPE_LIBRARY_SUFFIX so)
elseif(WIN32)
    set(FREETYPE_LIBRARY_SUFFIX lib)
endif()

macro(freetype_cmake_set_library_prefix libpng bzip2 brotli harfbuzz freetype)
    set(PNG_LIBRARY ${libpng}/libpng.${FREETYPE_LIBRARY_SUFFIX} ${libpng}/libpng16d.${FREETYPE_LIBRARY_SUFFIX})
    set(PNG_LIBRARIES ${libpng}/libpng.${FREETYPE_LIBRARY_SUFFIX} ${libpng}/libpng16d.${FREETYPE_LIBRARY_SUFFIX})
    set(BZIP2_LIBRARIES ${bzip2}/libbz2.${FREETYPE_LIBRARY_SUFFIX})
    set(BROTLIDEC_LIBRARIES ${brotli}/libbrotlidec.${FREETYPE_LIBRARY_SUFFIX} ${brotli}/libbrotlienc.${FREETYPE_LIBRARY_SUFFIX} ${brotli}/libbrotlicommon.${FREETYPE_LIBRARY_SUFFIX})
    set(HARFBUZZ_LIBRARIES ${harfbuzz}/libharfbuzz.${FREETYPE_LIBRARY_SUFFIX} ${harfbuzz}/libharfbuzz-subset.${FREETYPE_LIBRARY_SUFFIX})
    set(FREETYPE_LIBRARY ${freetype}/libfreetyped.${FREETYPE_LIBRARY_SUFFIX})
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

    set(BUILD_SHARED_LIBS ON CACHE BOOL "" FORCE)

    set(PNG_SHARED ON CACHE BOOL "" FORCE)
    set(PNG_STATIC OFF CACHE BOOL "" FORCE)
    set(PNG_EXECUTABLES OFF CACHE BOOL "" FORCE)
    set(PNG_TESTS OFF CACHE BOOL "" FORCE)
    set(SKIP_INSTALL_ALL OFF CACHE BOOL "" FORCE)

    set(HB_HAVE_FREETYPE OFF CACHE BOOL "" FORCE)

    set(BROTLI_DISABLE_TESTS ON CACHE BOOL "" FORCE)

    set(FT_WITH_BZIP2 ON CACHE BOOL "" FORCE)
    set(FT_WITH_PNG ON CACHE BOOL "" FORCE)
    set(FT_WITH_BROTLI ON CACHE BOOL "" FORCE)
    set(FT_WITH_HARFBUZZ ON CACHE BOOL "" FORCE)

    # build libpng
    add_subdirectory(${path_to_self}/libpng ${CMAKE_CURRENT_BINARY_DIR}/libpng)

    set(PNG_PNG_INCLUDE_DIR ${path_to_self}/libpng)
    include_directories(${PNG_PNG_INCLUDE_DIR})
    # generated files from libpng during build
    include_directories(${CMAKE_CURRENT_BINARY_DIR}/libpng)

    # build bzip2
    add_subdirectory(${path_to_self}/bzip2 ${CMAKE_CURRENT_BINARY_DIR}/bzip2)

    set(BZIP2_INCLUDE_DIR ${path_to_self}/bzip2)
    include_directories(${BZIP2_INCLUDE_DIR})

    # build brotlidec
    add_subdirectory(${path_to_self}/brotli ${CMAKE_CURRENT_BINARY_DIR}/brotli)

    set(BROTLIDEC_INCLUDE_DIRS ${path_to_self}/brotli/c/include)
    include_directories(${BROTLIDEC_INCLUDE_DIRS})

    # build harfbuzz
    add_subdirectory(${path_to_self}/harfbuzz ${CMAKE_CURRENT_BINARY_DIR}/harfbuzz)

    set(HARFBUZZ_INCLUDE_DIRS ${path_to_self}/harfbuzz/src)
    include_directories(${HARFBUZZ_INCLUDE_DIRS})

    if (CMAKE_LIBRARY_OUTPUT_DIRECTORY)
        freetype_cmake_set_library_prefixAll(${CMAKE_LIBRARY_OUTPUT_DIRECTORY})
    else()
        freetype_cmake_set_library_prefix(
                ${CMAKE_CURRENT_BINARY_DIR}/libpng
                ${CMAKE_CURRENT_BINARY_DIR}/bzip2
                ${CMAKE_CURRENT_BINARY_DIR}/brotli
                ${CMAKE_CURRENT_BINARY_DIR}/harfbuzz
                ${CMAKE_CURRENT_BINARY_DIR}/freetype-2.10.4
        )
    endif()

    add_subdirectory(${path_to_self}/freetype-2.10.4 ${CMAKE_CURRENT_BINARY_DIR}/freetype-2.10.4)
    set(FREETYPE_INCLUDE_DIRS ${path_to_self}/freetype-2.10.4/include)
    include_directories(${FREETYPE_INCLUDE_DIRS})

    # build freetype
    add_library(FREETYPE_CMAKE_LIB_FREETYPE SHARED ${path_to_self}/empty.cpp)

    add_dependencies(bz2 png)
    add_dependencies(brotlicommon bz2)
    add_dependencies(brotlienc brotlicommon)
    add_dependencies(brotlidec brotlienc)
    add_dependencies(harfbuzz brotlidec)
    add_dependencies(harfbuzz-subset harfbuzz)
    add_dependencies(freetype harfbuzz-subset)

    target_link_libraries(
            FREETYPE_CMAKE_LIB_FREETYPE
            png
            bz2
            brotlicommon
            brotlienc
            brotlidec
            harfbuzz-subset
            harfbuzz
            freetype
    )

    # override FREETYPE_LIBRARY
    if (CMAKE_LIBRARY_OUTPUT_DIRECTORY)
        set(
                FREETYPE_LIBRARY
                ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/libFREETYPE_CMAKE_LIB_FREETYPE.${FREETYPE_LIBRARY_SUFFIX}
        )
    else()
        set(
                FREETYPE_LIBRARY
                ${CMAKE_CURRENT_BINARY_DIR}/libFREETYPE_CMAKE_LIB_FREETYPE.${FREETYPE_LIBRARY_SUFFIX}
        )
    endif()
endmacro()

get_directory_property(FREETYPE_CMAKE_IS_SUB_DIRECTORY PARENT_DIRECTORY)
if (NOT FREETYPE_CMAKE_IS_SUB_DIRECTORY)
    freetype_cmake_include_freetype(${CMAKE_CURRENT_SOURCE_DIR})
    add_library(FREETYPE_CMAKE_TEST_LIBRARY empty.cpp)
    target_link_libraries(FREETYPE_CMAKE_TEST_LIBRARY FREETYPE_CMAKE_LIB_FREETYPE)
endif()