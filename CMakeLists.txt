cmake_minimum_required(VERSION 3.4)
project(App)

# build freetype

message(STATUS "png compile")
include_directories(libpng)
add_subdirectory(libpng)
message(STATUS "png complete")


message(STATUS "brotli compile")
include_directories(brotli)
add_subdirectory(brotli)
message(STATUS "brotli complete")


message(STATUS "harfbuzz compile")
include_directories(harfbuzz)
add_subdirectory(harfbuzz)
message(STATUS "harfbuzz complete")


message(STATUS "freetype compile")
include_directories(freetype-2.10.4)
add_subdirectory(freetype-2.10.4)
message(STATUS "freetype complete")

# test builder makes everything easy cus fk remembering cmake
add_subdirectory(testBuilder)

testBuilder_add_source(test empty.cpp)
testBuilder_add_library(test libpng)
testBuilder_add_library(test harfbuzz)
testBuilder_add_library(test freetype)
testBuilder_build_shared_library(test)