# The following variable can be defined on the command line:
#
#   BUILD_SHARED_LIBS
#
# The following environment variables will be taken into account when running
# cmake for the first time:
#
#   CFLAGS
#   LDFLAGS

cmake_minimum_required(VERSION 3.1)
project(ls-qpack LANGUAGES C)

option(LSQPACK_TESTS "Build tests")
option(LSQPACK_BIN "Build binaries" ON)
option(LSQPACK_XXH "Include XXH" ON)

# Use `cmake -DBUILD_SHARED_LIBS=OFF` to build a static library.
add_library(ls-qpack "")
target_include_directories(ls-qpack PUBLIC .)
target_sources(ls-qpack PRIVATE lsqpack.c)

target_include_directories(ls-qpack PRIVATE deps/xxhash/)
if(LSQPACK_XXH)
    target_sources(ls-qpack PRIVATE deps/xxhash/xxhash.c)
endif()

if(MSVC)
    target_include_directories(ls-qpack PUBLIC wincompat)
endif()

if(MSVC)
    target_compile_options(ls-qpack PRIVATE
        /Wall
        /wd4100 # unreffed parameter
        /wd4200 # zero-sized array
        # Apparently this C99 construct is not supported properly by VS:
        #   https://stackoverflow.com/questions/1064930/struct-initializer-typedef-with-visual-studio
        /wd4204 # non-constant aggregate initializer
        /wd4255 # no function prototype (getopt)
        /wd4334 # result of 32-bit shift implicitly converted to 64 bits
        /wd4820 # padding
        /wd4668 # undefined macro
        /wd4710 # not inlined by default
        /wd4996 # unsafe function
        /wd4061 # enum is not explicitly handled by a case
        /wd5045 # Compiler will insert Spectre mitigation for memory load
    )
else()
    target_compile_options(ls-qpack PRIVATE
        -Wall
        -Wextra
        -Wno-unused-parameter
        -fno-omit-frame-pointer
    )
endif()

IF (CMAKE_SYSTEM_NAME STREQUAL Windows)
    IF (LSQPACK_TESTS OR LSQPACK_BIN)
        FIND_PATH(GETOPT_INCLUDE_DIR NAMES getopt.h)
        IF (GETOPT_INCLUDE_DIR)
            INCLUDE_DIRECTORIES(${GETOPT_INCLUDE_DIR})
        ELSE()
            MESSAGE(FATAL_ERROR "getopt.h was not found")
        ENDIF()
            FIND_LIBRARY(GETOPT_LIB getopt)
        IF(GETOPT_LIB)
            MESSAGE(STATUS "Found getopt: ${GETOPT_LIB}")
        ELSE()
            MESSAGE(STATUS "getopt not found")
        ENDIF()
    ENDIF()
ENDIF()

IF(DEFINED LSXPACK_MAX_STRLEN)
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DLSXPACK_MAX_STRLEN=${LSXPACK_MAX_STRLEN}")
ENDIF()

IF (CMAKE_BUILD_TYPE STREQUAL MinSizeRel)
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DLS_QPACK_USE_LARGE_TABLES=0")
ENDIF()

IF(LSQPACK_TESTS)
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DLSQPACK_DEVEL_MODE=1")
ENDIF()

INCLUDE(CheckCCompilerFlag)
CHECK_C_COMPILER_FLAG(-Wno-implicit-fallthrough HAS_NO_IMPLICIT_FALLTHROUGH)
IF (HAS_NO_IMPLICIT_FALLTHROUGH)
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-implicit-fallthrough")
ENDIF()

SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} $ENV{EXTRA_CFLAGS}")
MESSAGE(STATUS "Compiler flags: ${CMAKE_C_FLAGS}")

if(LSQPACK_TESTS)
    enable_testing()
    add_subdirectory(test)
endif()

if(LSQPACK_BIN)
    add_subdirectory(bin)
endif()

install(TARGETS ls-qpack)
install(FILES lsqpack.h lsxpack_header.h DESTINATION include)
if(MSVC)
    install(DIRECTORY wincompat/sys DESTINATION include)
endif()
