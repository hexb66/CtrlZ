cmake_minimum_required(VERSION 3.14 FATAL_ERROR)
project(CtrlZ)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if(UNIX)
    set(CMAKE_THREAD_LIBS_INIT "-lpthread")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")
endif()

set(CMAKE_PREFIX_PATH ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty)

option(USE_ONNXRUNTIME_GPU "Use ONNX Runtime GPU version (requires providers_shared)" OFF)

find_package(nlohmann_json REQUIRED)
#aux_source_directory(Schedulers SRC_SCHEDULERS)
#aux_source_directory(Workers SRC_WORKERS)
#aux_source_directory(Utils SRC_UTILS)

file(GLOB_RECURSE SRC_SCHEDULERS Schedulers/*.cpp Schedulers/*.h)
file(GLOB_RECURSE SRC_WORKERS Workers/*.cpp Workers/*.h)
file(GLOB_RECURSE SRC_UTILS Utils/*.cpp Utils/*.h)


message("
Ctrl-Z is a muti-thread RL deployment framework for Robot locomotion.")
message("Visit http://opensource.zzshub.cn/CtrlZ for detailed documentation.")

message("+----------------------------------------+
|  CCCC  TTTTT  RRRRR  L           ZZZZZ |
| C        T    R   R  L              Z  |
| C        T    RRRRR  L    =====    Z   |
| C        T    R R    L            Z    |
|  CCCC    T    R  RR  LLLLL       ZZZZZ |
+----------------------------------------+
")

# include directory for all target
add_library(CtrlZ STATIC
    ${SRC_SCHEDULERS}
    ${SRC_WORKERS}
    ${SRC_UTILS}
)

add_subdirectory(Extensions)

target_include_directories(CtrlZ PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/include>
    $<INSTALL_INTERFACE:include/CtrlZ>
    $<INSTALL_INTERFACE:include/CtrlZ/ThirdParty/include>
)

if(WIN32)
    message("Build on windows")
    target_include_directories(CtrlZ PRIVATE ${ONNXRUNTIME_ROOT}/include)
    message("${ONNXRUNTIME_ROOT}/include")
    if(USE_ONNXRUNTIME_GPU)
        message(STATUS "Using ONNX Runtime GPU version")
        target_link_libraries(CtrlZ PRIVATE
            ${ONNXRUNTIME_ROOT}/lib/onnxruntime_providers_shared.lib
            ${ONNXRUNTIME_ROOT}/lib/onnxruntime.lib)
    else()
        message(STATUS "Using ONNX Runtime CPU version")
        target_link_libraries(CtrlZ PRIVATE
            ${ONNXRUNTIME_ROOT}/lib/onnxruntime.lib)
    endif()
elseif(UNIX)
    message("Build on unix like system")
    target_include_directories(CtrlZ PRIVATE ${ONNXRUNTIME_ROOT}/include)
    message("${ONNXRUNTIME_ROOT}/include")
    if(USE_ONNXRUNTIME_GPU)
        message(STATUS "Using ONNX Runtime GPU version")
        target_link_libraries(CtrlZ PRIVATE
            ${ONNXRUNTIME_ROOT}/lib/libonnxruntime_providers_shared.so
            ${ONNXRUNTIME_ROOT}/lib/libonnxruntime.so)
    else()
        message(STATUS "Using ONNX Runtime CPU version")
        target_link_libraries(CtrlZ PRIVATE
            ${ONNXRUNTIME_ROOT}/lib/libonnxruntime.so)
    endif()
else()
    message(FATAL_ERROR "Unsupported platform")
endif()

#set(READERWRITERQUEUE_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty)
#target_include_directories(CtrlZ PUBLIC ${READERWRITERQUEUE_INCLUDE_DIRS})
target_link_libraries(CtrlZ PUBLIC
    nlohmann_json::nlohmann_json
)

# Set target properties for installation
set_target_properties(CtrlZ PROPERTIES
    EXPORT_NAME CtrlZ
)

# Installation configuration
if(NOT DEFINED CtrlZ_VERSION)
    set(CtrlZ_VERSION "1.0.0")
endif()

# Display installation prefix
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    if(DEFINED ENV{CONDA_PREFIX})
        message(STATUS "Conda environment detected: $ENV{CONDA_PREFIX}")
        message(STATUS "Installation prefix: ${CMAKE_INSTALL_PREFIX}")
        message(STATUS "  Libraries will be installed to: ${CMAKE_INSTALL_PREFIX}/lib")
        message(STATUS "  Headers will be installed to: ${CMAKE_INSTALL_PREFIX}/include/CtrlZ")
        message(STATUS "  CMake config will be installed to: ${CMAKE_INSTALL_PREFIX}/lib/cmake/CtrlZ")
    else()
        message(STATUS "Installation prefix: ${CMAKE_INSTALL_PREFIX}")
        message(STATUS "  Libraries will be installed to: ${CMAKE_INSTALL_PREFIX}/lib")
        message(STATUS "  Headers will be installed to: ${CMAKE_INSTALL_PREFIX}/include/CtrlZ")
        message(STATUS "  CMake config will be installed to: ${CMAKE_INSTALL_PREFIX}/lib/cmake/CtrlZ")
        message(STATUS "  To change installation prefix, use: cmake -DCMAKE_INSTALL_PREFIX=<path>")
    endif()
else()
    message(STATUS "Installation prefix (custom): ${CMAKE_INSTALL_PREFIX}")
endif()

install(TARGETS CtrlZ
    EXPORT CtrlZTargets
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin
    INCLUDES DESTINATION include
)

install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/
    DESTINATION include/CtrlZ
    FILES_MATCHING
    PATTERN "*.hpp"
    PATTERN "*.h"
    PATTERN "CMakeLists.txt" EXCLUDE
    PATTERN "*.cmake" EXCLUDE
)

install(EXPORT CtrlZTargets
    FILE CtrlZTargets.cmake
    NAMESPACE CtrlZ::
    DESTINATION lib/cmake/CtrlZ
)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# Convert boolean to 0/1 for robust CMake if() evaluation in generated config
if(USE_ONNXRUNTIME_GPU)
    set(USE_ONNXRUNTIME_GPU_VALUE 1)
else()
    set(USE_ONNXRUNTIME_GPU_VALUE 0)
endif()

# Generate CtrlZConfig.cmake (avoid configure_package_config_file here; keep it simple and stable)
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/CtrlZConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/CtrlZConfig.cmake
    @ONLY
)

write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/CtrlZConfigVersion.cmake
    VERSION ${CtrlZ_VERSION}
    COMPATIBILITY SameMajorVersion
)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/CtrlZConfig.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/CtrlZConfigVersion.cmake
    DESTINATION lib/cmake/CtrlZ
)