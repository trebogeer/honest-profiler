##########################################################
# CMake Configuration

cmake_minimum_required(VERSION 2.8 FATAL_ERROR)

project(honest_profiler)

enable_testing()

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "build")

include(CheckLibraryExists)
include(CheckSymbolExists)
include(CheckFunctionExists)

##########################################################
# Paths
set(SRC "src/main/cpp")
set(SRC_TEST "src/test/cpp")
set(BIN "build")
set(OUTPUT "lagent")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${BIN})

##########################################################
# Libraries & Code

find_package(PkgConfig REQUIRED)

find_package(JNI REQUIRED)
include_directories(${JAVA_INCLUDE_PATH})
include_directories(${JAVA_INCLUDE_PATH2})

if (DEFINED ENV{UNITTEST_INCLUDE_DIRS})
    message("User has configured " $ENV{UNITTEST_INCLUDE_DIRS} " as the unit test include directory")
    file(GLOB_RECURSE UNIT_TEST_H $ENV{UNITTEST_INCLUDE_DIRS}/*UnitTest++.h)
else()
    if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
        find_package(UnitTest++ REQUIRED NO_MODULE)
        include_directories(${UTPP_INCLUDE_DIRS})
    else()
        pkg_check_modules (unittest++ REQUIRED unittest++)
        include_directories(${unittest++_INCLUDE_DIRS})
    endif()
    file(GLOB_RECURSE UNIT_TEST_H ${unittest++_INCLUDE_DIRS}/*UnitTest++.h)
endif()

get_filename_component(UNIT_TEST_INCL ${UNIT_TEST_H} DIRECTORY)
include_directories(${UNIT_TEST_INCL})

# include of gcc 4.8 headers specifically to work around
# https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=729933
# equivalent in ubuntu 13.10
include_directories("/usr/include/i386-linux-gnu/c++/4.8/")

set(SOURCE_FILES
    ${SRC}/agent.cpp
    ${SRC}/circular_queue.cpp
    ${SRC}/circular_queue.h
    ${SRC}/common.cpp
    ${SRC}/common.h
    ${SRC}/control.cpp
    ${SRC}/controller.cpp
    ${SRC}/controller.h
    ${SRC}/globals.h
    ${SRC}/log_writer.cpp
    ${SRC}/log_writer.h
    ${SRC}/signal_handler.cpp
    ${SRC}/signal_handler.h
    ${SRC}/processor.cpp
    ${SRC}/processor.h
    ${SRC}/profiler.cpp
    ${SRC}/profiler.h
    ${SRC}/stacktraces.h
    ${SRC}/trace.h
    ${SRC}/thread_map.h
    ${SRC}/thread_map.cpp
    ${SRC}/concurrent_map.h
    ${SRC}/concurrent_map.cpp)

set(TEST_FILES
    ${SRC_TEST}/fixtures.h
    ${SRC_TEST}/test_circular_queue.cpp
    ${SRC_TEST}/test.cpp
    ${SRC_TEST}/test_log_writer.cpp
    ${SRC_TEST}/test_agent.cpp
    ${SRC_TEST}/test.h
    ${SRC_TEST}/test_profiler_config.cpp
    ${SRC_TEST}/test_maps.cpp)


##########################################################
# Compiler Options

set(GLOBAL_WARNINGS "-Wall -Wformat-security -Wno-char-subscripts")
set(GLOBAL_WARNINGS "${GLOBAL_WARNINGS} -Wno-sign-compare -Wno-strict-overflow -Wwrite-strings")
set(GLOBAL_WARNINGS "${GLOBAL_WARNINGS} -Wnon-virtual-dtor -Woverloaded-virtual")

set(GLOBAL_COPTS "-fdiagnostics-show-option -fno-omit-frame-pointer -fno-strict-aliasing")
set(GLOBAL_COPTS "${GLOBAL_COPTS} -funsigned-char -fno-asynchronous-unwind-tables -msse2 -g -D__STDC_FORMAT_MACROS -O2")

# Platform Specific
if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(GLOBAL_COPTS "${GLOBAL_COPTS} -DTARGET_RT_MAC_CFM=0")
    if("${CMAKE_SIZEOF_VOID_P}" EQUAL "8")
        message("-- Detected 64 bit Mac")
        set(GLOBAL_COPTS "${GLOBAL_COPTS} -D__LP64__=1")
    endif()
endif()

# Compiler Specific
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
    message("-- using Clang")
    set(GLOBAL_WARNINGS, "${GLOBAL_WARNINGS} -Weverything -Wno-c++98-compat-pedantic -Wno-padded")
    set(global_warNINGS, "${GLOBAL_WARNINGS} -Wno-missing-prototypes -Wno-weak-vtables -Wno-global-constructors")
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    message("-- using GCC")
    #set(GLOBAL_COPTS "${GLOBAL_COPTS} -mfpmath=sse -fno-tree-loop-distribute-patterns")
    set(GLOBAL_COPTS "${GLOBAL_COPTS} -mfpmath=sse")
    set(GLOBAL_WARNINGS, "${GLOBAL_WARNINGS} -Wframe-larger-than=16384 -Wno-unused-but-set-variable")
    set(GLOBAL_WARNINGS, "${GLOBAL_WARNINGS} -Wunused-but-set-parameter -Wvla -Wno-conversion-null")
    set(GLOBAL_WARNINGS, "${GLOBAL_WARNINGS} -Wno-unknown-pragmas -Wno-builtin-macro-redefined -Wl,-fatal_warnings")
endif()

# to enable tracing add: -DENABLE_TRACING
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${GLOBAL_WARNINGS} ${GLOBAL_COPTS} -pthread -std=c++0x $ENV{EXT_CXX_FLAGS}")

##########################################################
# Build

add_library(${OUTPUT} SHARED ${SOURCE_FILES})

if ("${CMAKE_SYSTEM_NAME}" STREQUAL "FreeBSD") 
    target_link_libraries(${OUTPUT} ${JAVA_JVM_LIBRARY} rt)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    target_link_libraries(${OUTPUT} ${JAVA_JVM_LIBRARY} dl)
else()
    target_link_libraries(${OUTPUT} ${JAVA_JVM_LIBRARY} dl rt)
endif()

add_executable(unitTests ${UNIT_TEST_H} ${TEST_FILES})

if (DEFINED ENV{UNITTEST_LIBRARIES})
    message("User has configured " $ENV{UNITTEST_LIBRARIES} " as the unit test libraries")
    target_link_libraries(unitTests ${OUTPUT} $ENV{UNITTEST_LIBRARIES})
else()
    if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
        target_link_libraries(unitTests ${OUTPUT} UnitTest++)
    else()
        target_link_libraries(unitTests ${OUTPUT} ${unittest++_LIBRARIES})
    endif()
endif()

# make test
add_test(unitTests ${BIN}/unitTests)
