cmake_minimum_required(VERSION 3.4)
project(FREETYPE_CMAKE)

macro(freetype_cmake_set_library_prefix libpng bzip2 brotli harfbuzz freetype)
    set(PNG_LIBRARY ${libpng}/libpng.a ${libpng}/libpng16d.a)
    set(PNG_LIBRARIES ${libpng}/libpng.a ${libpng}/libpng16d.a)
    set(BZIP2_LIBRARIES ${bzip2}/libbz2d.a)
    set(BROTLIDEC_LIBRARIES ${brotli}/libbrotlidec-static.a ${brotli}/libbrotlienc-static.a ${brotli}/libbrotlicommon-static.a)
    set(HARFBUZZ_LIBRARIES ${harfbuzz}/libharfbuzz.a ${harfbuzz}/libharfbuzz-subset.a)
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

    set(PNG_SHARED OFF CACHE BOOL "" FORCE)
    set(PNG_STATIC ON CACHE BOOL "" FORCE)
    set(PNG_EXECUTABLES OFF CACHE BOOL "" FORCE)
    set(PNG_TESTS OFF CACHE BOOL "" FORCE)
    set(SKIP_INSTALL_ALL OFF CACHE BOOL "" FORCE)

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

    if (CMAKE_LIBRARY_OUTPUT_DIRECTORY)
        freetype_cmake_set_library_prefixAll(${CMAKE_LIBRARY_OUTPUT_DIRECTORY})
    else()
        freetype_cmake_set_library_prefix(libpng bzip2 brotli harfbuzz freetype-2.10.4)
    endif()

    add_subdirectory(${path_to_self}/freetype-2.10.4 ${CMAKE_CURRENT_BINARY_DIR}/freetype-2.10.4)
    set(FREETYPE_INCLUDE_DIRS ${path_to_self}/freetype-2.10.4/include)
    include_directories(${FREETYPE_INCLUDE_DIRS})

    # build harfbuzz
    add_library(FREETYPE_CMAKE_LIB_FREETYPE_BASE ${path_to_self}/empty.cpp)

    add_dependencies(libz2 png_static)
    add_dependencies(brotlicommon-static libz2)
    add_dependencies(brotlienc-static brotlicommon-static)
    add_dependencies(brotlidec-static brotlienc-static)
    add_dependencies(freetype brotlidec-static)

    target_link_libraries(
            FREETYPE_CMAKE_LIB_FREETYPE_BASE STATIC
            png_static
            libz2
            brotlicommon-static
            brotlienc-static
            brotlidec-static
            freetype
    )

    add_subdirectory(${path_to_self}/harfbuzz ${CMAKE_CURRENT_BINARY_DIR}/harfbuzz)

    set(HARFBUZZ_INCLUDE_DIRS ${path_to_self}/harfbuzz/src)
    include_directories(${HARFBUZZ_INCLUDE_DIRS})

    # build
    add_library(FREETYPE_CMAKE_LIB_FREETYPE ${path_to_self}/empty.cpp)

    # harfbuzz builds before harfbuzz-subset
    add_dependencies(harfbuzz FREETYPE_CMAKE_LIB_FREETYPE_BASE)

    target_link_libraries(
            FREETYPE_CMAKE_LIB_FREETYPE STATIC
            FREETYPE_CMAKE_LIB_FREETYPE_BASE
            harfbuzz-subset
            harfbuzz
    )
    if (CMAKE_LIBRARY_OUTPUT_DIRECTORY)
        # override FREETYPE_LIBRARY
        set(
                FREETYPE_LIBRARY
                ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/libFREETYPE_CMAKE_LIB_FREETYPE.a
        )
    else()
        freetype_cmake_set_library_prefix(
                ${CMAKE_CURRENT_BINARY_DIR}/libpng
                ${CMAKE_CURRENT_BINARY_DIR}/bzip2
                ${CMAKE_CURRENT_BINARY_DIR}/brotli
                ${CMAKE_CURRENT_BINARY_DIR}/harfbuzz
                ${CMAKE_CURRENT_BINARY_DIR}/freetype-2.10.4
        )
        # override FREETYPE_LIBRARY
        set(
                FREETYPE_LIBRARY
                ${CMAKE_CURRENT_BINARY_DIR}/libFREETYPE_CMAKE_LIB_FREETYPE.a
        )
    endif()
endmacro()

get_directory_property(FREETYPE_CMAKE_IS_SUB_DIRECTORY PARENT_DIRECTORY)
if (NOT FREETYPE_CMAKE_IS_SUB_DIRECTORY)
    freetype_cmake_include_freetype(${CMAKE_CURRENT_SOURCE_DIR})
    add_library(FREETYPE_CMAKE_TEST_LIBRARY empty.cpp)
    target_link_libraries(FREETYPE_CMAKE_TEST_LIBRARY FREETYPE_CMAKE_LIB_FREETYPE)
endif()