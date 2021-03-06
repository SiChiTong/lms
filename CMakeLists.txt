cmake_minimum_required(VERSION 2.8)
project(LMS C CXX)

#conan setup
include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
conan_basic_setup()

option(BUILD_TESTS "Build tests" OFF)
option(COVERAGE "Enable test coverage" OFF)

find_package(Threads REQUIRED)

include_directories(${CMAKE_CURRENT_BINARY_DIR}/include)
include_directories(include)
include_directories(${ProtobufIncludePath})
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR})

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
include(TestCpp11)

# Protobuf
#find_package(Protobuf REQUIRED)
#file(GLOB ProtoFiles "${CMAKE_CURRENT_SOURCE_DIR}/messages/*.proto")
#PROTOBUF_GENERATE_CPP(ProtoSources ProtoHeaders ${ProtoFiles})

include(CMakeData.txt)

#LMS is standalone
set(LMS_DIR_CONFIGS "")
set(LMS_DIR_MODULES "")
set(LMS_DIR_EXTERNAL "")

if((${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang") OR (${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU"))
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -Wextra -Wpedantic")
endif()

#create definitions.h
configure_file(include/lms/definitions.h.in
    "${CMAKE_CURRENT_BINARY_DIR}/include/lms/definitions.h"
    ESCAPE_QUOTES)

set (HEADERS
    "${CMAKE_CURRENT_BINARY_DIR}/include/lms/definitions.h"

    "include/lms/logger.h"
    "include/lms/module.h"
    "include/lms/lms_exports.h"
    "include/lms/messaging.h"
    "include/lms/serializable.h"
    "include/lms/deprecated.h"
    "include/lms/config.h"
    "include/lms/inheritance.h"
    "include/lms/service_handle.h"
    "include/lms/service.h"
    "include/lms/endian.h"
    "include/lms/unit.h"
    "include/lms/interface.h"
    "include/lms/life_cycle.h"
    "include/lms/type.h"
    "include/lms/time.h"
    "include/lms/datamanager.h"
    "include/lms/exception.h"
    "include/lms/client.h"
    "include/lms/protobuf_socket.h"

    "src/internal/framework.h"
    "src/internal/signalhandler.h"
    "src/internal/colors.h"
    "src/internal/argumenthandler.h"
    "src/internal/executionmanager.h"
    "src/internal/loader.h"
    "src/internal/xml_parser.h"
    "src/internal/file_monitor.h"
    "src/internal/dag.h"
    "src/internal/debug_server.h"
    "src/internal/watch_dog.h"
    "src/internal/string.h"
    "src/internal/backtrace_formatter.h"
    "src/internal/dot_exporter.h"
    "src/internal/module_channel_graph.h"
    "src/internal/viz.h"
    "src/internal/daemon.h"
    "src/internal/master.h"
    "src/internal/clock.h"
    "src/internal/os.h"
    "src/internal/protobuf_sink.h"
    "src/internal/test_modules.h"
    "src/internal/profiler.h"

    "include/lms/logging/level.h"
    "include/lms/logging/logger.h"
    "include/lms/logging/event.h"
    "include/lms/logging/sink.h"
    "include/lms/logging/filter.h"
    "include/lms/logging/context.h"
    "include/lms/logging/console_sink.h"
    "include/lms/logging/file_sink.h"
    "include/lms/logging/threshold_filter.h"
    "include/lms/logging/trace.h"
)

set (SOURCE
    "src/module.cpp"
    "src/messaging.cpp"
    "src/config.cpp"
    "src/service.cpp"
    "src/endian.cpp"
    "src/unit.cpp"
    "src/type.cpp"
    "src/time.cpp"
    "src/datamanager.cpp"
    "src/client.cpp"
    "src/protobuf_socket.cpp"

    "src/internal/executionmanager.cpp"
    "src/internal/framework.cpp"
    "src/internal/signalhandler.cpp"
    "src/internal/argumenthandler.cpp"
    "src/internal/file_monitor.cpp"
    "src/internal/debug_server.cpp"
    "src/internal/watch_dog.cpp"
    "src/internal/loader.cpp"
    "src/internal/backtrace_formatter.cpp"
    "src/internal/loader.cpp"
    "src/internal/signalhandler.cpp"
    "src/internal/clock.cpp"
    "src/internal/xml_parser.cpp"
    "src/internal/string.cpp"
    "src/internal/os.cpp"
    "src/internal/dot_exporter.cpp"
    "src/internal/viz.cpp"
    "src/internal/daemon.cpp"
    "src/internal/master.cpp"
    "src/internal/protobuf_sink.cpp"
    "src/internal/test_modules.cpp"
    "src/internal/profiler.cpp"

    "src/logging/level.cpp"
    "src/logging/logger.cpp"
    "src/logging/context.cpp"
    "src/logging/event.cpp"
    "src/logging/console_sink.cpp"
    "src/logging/file_sink.cpp"
    "src/logging/threshold_filter.cpp"
)

if(POLICY CMP0042)
    # Set new MACOSX_RPATH policy on OS X
    cmake_policy(SET CMP0042 NEW)
endif()

# LMS core library
add_library (lmscore SHARED
    ${HEADERS}
    ${SOURCE}
    "include/messages.pb.cc"
    "include/recording_meta.pb.cc"
    ${ProtoSources} ${ProtoHeaders}
)

if(COVERAGE)
    message(STATUS "Enable test coverage")
    set_target_properties(lmscore PROPERTIES COMPILE_FLAGS "-fprofile-arcs -ftest-coverage -O0")
    set_target_properties(lmscore PROPERTIES LINK_FLAGS "-fprofile-arcs")
endif()

# Compile xml-parser pugixml

target_link_libraries(lmscore PRIVATE pugixml ${PROTOBUF_LIBRARY})

# System specific link
# http://www.openguru.com/2009/04/cmake-detecting-platformoperating.html
if(UNIX)
    target_link_libraries(lmscore PRIVATE dl)
endif()
target_link_libraries(lmscore PRIVATE ${CMAKE_THREAD_LIBS_INIT} ${CONAN_LIBS})

# LMS executable
add_executable(lms
    src/main.cpp
)
target_link_libraries(lms PRIVATE lmscore)

#testing
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/test")#TODO better handling
    message(STATUS "enabling tests")
    enable_testing()
    add_subdirectory(test)
else(EXISTS test)
    message(STATUS "no tests available")
endif()


add_custom_target(format clang-format -i ${HEADERS} ${SOURCE}
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
