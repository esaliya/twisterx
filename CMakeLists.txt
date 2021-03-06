##########################################################################
#	CMake Build Rules for the TwisterX                               #
##########################################################################
# Basic Usage:                                                           #
#   cmake .								 #
#   make  								 #
#   make test                                                            #
# For more information about CMake, see http://www.cmake.org             #
##########################################################################
cmake_minimum_required(VERSION 3.5 FATAL_ERROR)

project(TWISTERX)

set(TWISTERX_VERSION 0.0)

message (STATUS "value: ${TWISTERX_SOURCE_DIR}" )

# So we can find our custom modules
list(APPEND CMAKE_MODULE_PATH ${TWISTERX_SOURCE_DIR}/CMake)

# Put the libaries and binaries that get built into directories at the
# top of the build.
SET(LIBRARY_OUTPUT_PATH ${CMAKE_BINARY_DIR}/lib)
SET(EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR}/bin)
SET(TEST_DATA_DIR ${TWISTERX_SOURCE_DIR}/data)

# Find MPI
find_package(MPI REQUIRED)
include_directories(${MPI_CXX_INCLUDE_PATH})

if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    message(STATUS "No build type selected, default to Debug")
    set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Build type (default Debug)" FORCE)
endif()

# C++ standard
SET(CMAKE_CXX_STANDARD 17)
SET(CMAKE_CXX_STANDARD_REQUIRED ON)
SET(CMAKE_CXX_EXTENSIONS OFF)

# compiler
#SET(CMAKE_CXX_COMPILER mpicxx)
#SET(CMAKE_C_COMPILER mpicc)

IF(NOT (CMAKE_BUILD_TYPE STREQUAL "Release" OR CMAKE_BUILD_TYPE STREQUAL "RELEASE"))
    SET(CMAKE_BUILD_TYPE "Debug")
ENDIF(NOT (CMAKE_BUILD_TYPE STREQUAL "Release" OR CMAKE_BUILD_TYPE STREQUAL "RELEASE"))

#Enable address sanitizer and disable return-value-optimizations
IF(CMAKE_BUILD_TYPE STREQUAL "Debug" OR CMAKE_BUILD_TYPE STREQUAL "DEBUG")
    SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=address -fno-elide-constructors")
ENDIF(CMAKE_BUILD_TYPE STREQUAL "Debug" OR CMAKE_BUILD_TYPE STREQUAL "DEBUG")

SET(CMAKE_REQUIRED_FLAGS "-std=c++17")

include(CheckCXXSourceCompiles)
check_cxx_source_compiles(
        "#include<numeric>
    int main() {
        int* ptr;
        std::exclusive_scan(ptr, ptr, ptr, 0);
    }"
        HAVE_EXCLUSIVE_SCAN
)

if(NOT HAVE_EXCLUSIVE_SCAN)
    add_definitions(-DNEED_EXCLUSIVE_SCAN)
endif()

#Off if you dont want to build tests -- ON default
option(TWISTERX_BUILD_TESTS "Build tests." ON)

#Reference -- https://github.com/ComicSansMS/GhulbusBase/blob/master/CMakeLists.txt
if(TWISTERX_BUILD_TESTS)
    enable_testing()

    if(NOT TARGET Catch)
        include(ExternalProject)
        if(IS_TRAVIS_BUILD)     # on travis, use git for fetching instead of wget
            set(TWISTERX_FETCH_EXTERNAL_CATCH
                    GIT_REPOSITORY https://github.com/philsquared/Catch.git
                    GIT_TAG 19ab2117c5bac2f376f8da4a4b25e183137bcec0)
        elseif(WIN32)
            set(TWISTERX_FETCH_EXTERNAL_CATCH
                    URL https://github.com/catchorg/Catch2/archive/v2.0.1.zip
                    URL_HASH MD5=1abca1b324b99b1631e999119b172620)
        else()
            set(TWISTERX_FETCH_EXTERNAL_CATCH
                    URL https://github.com/catchorg/Catch2/archive/v2.0.1.tar.gz
                    URL_HASH MD5=2080f4696579351d9323b3b5a8c3c71b)
        endif()
        ExternalProject_Add(Catch-External
                PREFIX ${CMAKE_BINARY_DIR}/external/Catch
                ${TWISTERX_FETCH_EXTERNAL_CATCH}
                CONFIGURE_COMMAND ""
                BUILD_COMMAND ""
                INSTALL_COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/external/Catch/src/Catch-External/single_include/catch.hpp
                ${CMAKE_BINARY_DIR}/external/Catch/include/catch.hpp
                )
        add_library(Catch INTERFACE)
        add_dependencies(Catch Catch-External)
        target_include_directories(Catch INTERFACE ${CMAKE_BINARY_DIR}/external/Catch/include)
        target_compile_definitions(Catch INTERFACE $<$<CXX_COMPILER_ID:MSVC>:_SILENCE_CXX17_UNCAUGHT_EXCEPTION_DEPRECATION_WARNING>)
    endif()
    include(CMake/Catch.cmake)
    add_subdirectory(test)
endif()




add_subdirectory(src)

