cmake_minimum_required(VERSION 3.4)
project(FREETYPE_CMAKE)

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

    add_dependencies(bz2 png)
    add_dependencies(brotlicommon bz2)
    add_dependencies(brotlienc brotlicommon)
    add_dependencies(brotlidec brotlienc)
    add_dependencies(harfbuzz brotlidec)
    add_dependencies(freetype harfbuzz)
    add_dependencies(FREETYPE_CMAKE_LIB_FREETYPE freetype)

    set(FREETYPE_CMAKE_LIB_FREETYPE_LIBS png bz2 brotlicommon brotlienc brotlidec)
    if (APPLE)
        target_link_libraries(
                FREETYPE_CMAKE_LIB_FREETYPE
                ${FREETYPE_CMAKE_LIB_FREETYPE_LIBS}
                "-framework CoreFoundation"
                "-framework CoreGraphics"
                "-framework CoreText"
                "-Wl,-all_load,${HARFBUZZ_LIBRARIES}"
                "-Wl,-all_load,${FREETYPE_LIBRARY}"
        )
    endif()

    # override FREETYPE_LIBRARY to point to full freetype library
    set(FREETYPE_LIBRARY ${FREETYPE_PREFEX_LIB_OUT_DIR}/libFREETYPE_CMAKE_LIB_FREETYPE.${FREETYPE_CMAKE_LIB_LIBRARY_SUFFIX})
endmacro()

get_directory_property(FREETYPE_CMAKE_IS_SUB_DIRECTORY PARENT_DIRECTORY)
if (NOT FREETYPE_CMAKE_IS_SUB_DIRECTORY)
    freetype_cmake_include_freetype(${CMAKE_CURRENT_SOURCE_DIR})
    add_library(FREETYPE_CMAKE_TEST_LIBRARY empty.cpp)
    target_link_libraries(FREETYPE_CMAKE_TEST_LIBRARY FREETYPE_CMAKE_LIB_FREETYPE)
endif()