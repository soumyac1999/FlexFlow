cmake_minimum_required(VERSION 3.10)
project(FlexFlow)

include(ExternalProject)

# Set policy CMP0074 to eliminate cmake warnings
cmake_policy(SET CMP0074 NEW)
cmake_policy(SET CMP0077 NEW)
if (CMAKE_VERSION VERSION_GREATER_EQUAL "3.24.0")
  # Fix DOWNLOAD_EXTRACT_TIMESTAMP warnings
  cmake_policy(SET CMP0135 NEW)
endif()
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_LIST_DIR}/cmake)
set(FLEXFLOW_ROOT ${CMAKE_CURRENT_LIST_DIR})
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -UNDEBUG")

# Set a default build type if none was specified
set(default_build_type "Debug")
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to '${default_build_type}' as none was specified.")
  set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE
      STRING "Choose the type of build." FORCE)
endif()

# do not disable assertions even if in release mode
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -UNDEBUG")

if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
  set(LIBEXT ".so")
endif()

# only used for pypi
option(FF_BUILD_FROM_PYPI "Build from pypi" OFF)

# build shared or static flexflow lib
option(BUILD_SHARED_LIBS "Build shared libraries instead of static ones" ON)

# option for using Python
option(FF_USE_PYTHON "Enable Python" ON)

# option to download pre-compiled NCCL/Legion libraries
option(FF_USE_PREBUILT_NCCL "Enable use of NCCL pre-compiled library, if available" ON)
option(FF_USE_PREBUILT_LEGION "Enable use of Legion pre-compiled library, if available" ON)
option(FF_USE_ALL_PREBUILT_LIBRARIES "Enable use of all pre-compiled libraries, if available" OFF)

# option for using Python
set(FF_GASNET_CONDUITS aries udp mpi ibv ucx)
set(FF_GASNET_CONDUIT "mpi" CACHE STRING "Select GASNet conduit ${FF_GASNET_CONDUITS}")
set_property(CACHE FF_GASNET_CONDUIT PROPERTY STRINGS ${FF_GASNET_CONDUITS})
set(FF_LEGION_NETWORKS "" CACHE STRING "Network backend(s) to use")

if ((FF_LEGION_NETWORKS STREQUAL "gasnet" AND FF_GASNET_CONDUIT STREQUAL "ucx") OR FF_LEGION_NETWORKS STREQUAL "ucx")
    if("${FF_UCX_URL}" STREQUAL "")
        set(UCX_URL "https://github.com/openucx/ucx/releases/download/v1.14.0-rc1/ucx-1.14.0.tar.gz")
    else()
        set(UCX_URL "${FF_UCX_URL}")
    endif()

    set(UCX_DIR ${CMAKE_CURRENT_BINARY_DIR}/ucx)
    get_filename_component(UCX_COMPRESSED_FILE_NAME "${UCX_URL}" NAME)
    # message(STATUS "UCX_URL: ${UCX_URL}")
    # message(STATUS "UCX_COMPRESSED_FILE_NAME: ${UCX_COMPRESSED_FILE_NAME}")
    set(UCX_COMPRESSED_FILE_PATH "${CMAKE_CURRENT_BINARY_DIR}/${UCX_COMPRESSED_FILE_NAME}")
    set(UCX_BUILD_NEEDED OFF)
    set(UCX_CONFIG_FILE ${UCX_DIR}/config.txt)
    set(UCX_BUILD_OUTPUT ${UCX_DIR}/build.log)

    if(EXISTS ${UCX_CONFIG_FILE})
        file(READ ${UCX_CONFIG_FILE} PREV_UCX_CONFIG)
        # message(STATUS "PREV_UCX_CONFIG: ${PREV_UCX_CONFIG}")
        if("${UCX_URL}" STREQUAL "${PREV_UCX_CONFIG}")
            # configs match - no build needed
            set(UCX_BUILD_NEEDED OFF)
        else()
	        message(STATUS "UCX configuration has changed - rebuilding...")
            set(UCX_BUILD_NEEDED ON)
        endif()
    else()
        message(STATUS "Configuring and building UCX...")
        set(UCX_BUILD_NEEDED ON)
    endif()

    if(UCX_BUILD_NEEDED)
        if(NOT EXISTS "${UCX_COMPRESSED_FILE_PATH}")
            message(STATUS "Downloading openucx/ucx from: ${UCX_URL}")
            file(
                DOWNLOAD
                "${UCX_URL}" "${UCX_COMPRESSED_FILE_PATH}"
                SHOW_PROGRESS
                STATUS status
                LOG log
            )

            list(GET status 0 status_code)
            list(GET status 1 status_string)

            if(status_code EQUAL 0)
                message(STATUS "Downloading... done")
            else()
                message(FATAL_ERROR "error: downloading '${UCX_URL}' failed
                    status_code: ${status_code}
                    status_string: ${status_string}
                    log:
                    --- LOG BEGIN ---
                    ${log}
                    --- LOG END ---"
                )
            endif()
        else()
            message(STATUS "${UCX_COMPRESSED_FILE_NAME} already exists")
        endif()

        execute_process(COMMAND mkdir -p ${UCX_DIR})
        execute_process(COMMAND tar xzf ${UCX_COMPRESSED_FILE_PATH} -C ${UCX_DIR} --strip-components 1)
        message(STATUS "Building UCX...")
        execute_process(
            COMMAND sh -c "cd ${UCX_DIR} && ${UCX_DIR}/contrib/configure-release --prefix=${UCX_DIR}/install --enable-mt && make -j8 && make install"
            RESULT_VARIABLE UCX_BUILD_STATUS
            OUTPUT_FILE ${UCX_BUILD_OUTPUT}
            ERROR_FILE ${UCX_BUILD_OUTPUT}
        )

        if(UCX_BUILD_STATUS)
            message(FATAL_ERROR "UCX build result = ${UCX_BUILD_STATUS} - see ${UCX_BUILD_OUTPUT} for more details")
        endif()

        # Currently, we use default build configurations for UCX and therefore only save URL as configuration settings
        file(WRITE ${UCX_CONFIG_FILE} "${UCX_URL}")
    endif()

    if (FF_LEGION_NETWORKS STREQUAL "gasnet" AND FF_GASNET_CONDUIT STREQUAL "ucx")
        set(ENV{UCX_HOME} "${UCX_DIR}/install")
    endif()

    if (FF_LEGION_NETWORKS STREQUAL "ucx")
        set(ucx_DIR ${UCX_DIR}/cmake)
        set(ENV{Legion_NETWORKS} "ucx")
        message(STATUS "Legion_NETWORKS: $ENV{Legion_NETWORKS}")
    endif()
else()
    message(STATUS "FF_GASNET_CONDUIT: ${FF_GASNET_CONDUIT}")
endif()

set(FF_GPU_BACKENDS cuda hip_cuda hip_rocm intel)
set(FF_GPU_BACKEND "cuda" CACHE STRING "Select GPU Backend ${FF_GPU_BACKENDS}")
set_property(CACHE FF_GPU_BACKEND PROPERTY STRINGS ${FF_GPU_BACKENDS})

# option for cuda arch
set(FF_CUDA_ARCH "autodetect" CACHE STRING "Target CUDA Arch")
if (FF_CUDA_ARCH STREQUAL "")
  message(FATAL_ERROR "FF_CUDA_ARCH cannot be an empty string. Set it to `autodetect`, `all`, or pass one or multiple valid CUDA archs.")
endif()

# option for nccl
option(FF_USE_NCCL "Run FlexFlow with NCCL" OFF)

if (FF_GPU_BACKEND STREQUAL "hip_rocm" AND FF_USE_NCCL STREQUAL "ON")
  message(FATAL_ERROR "NCCL: ON for FF_GPU_BACKEND: hip_rocm. hip_rocm backend must have NCCL disabled.")
endif()

# option for avx2
option(FF_USE_AVX2 "Run FlexFlow with AVX2" OFF)

# option for max dim
set(FF_MAX_DIM "4" CACHE STRING "Maximum dimention of tensors")

# option for legion
option(FF_USE_EXTERNAL_LEGION "Use pre-installed Legion" OFF)

set(FLEXFLOW_EXT_LIBRARIES "")
set(FLEXFLOW_INCLUDE_DIRS "")

# get FLAGS from ENV
set(CC_FLAGS $ENV{CC_FLAGS})
set(NVCC_FLAGS $ENV{NVCC_FLAGS})
set(LD_FLAGS $ENV{LD_FLAGS})

# Set global FLAGS
list(APPEND CC_FLAGS
  -std=c++11)

list(APPEND NVCC_FLAGS
  -std=c++11)

add_compile_options(${CC_FLAGS})
set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} ${NVCC_FLAGS})
link_libraries(${LD_FLAGS})

# Detect OS type and Linux version (if it applies)
set(LINUX_VERSION "")
if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
  find_program(LSB_RELEASE_EXEC lsb_release)
  if(LSB_RELEASE_EXEC)
    execute_process(COMMAND ${LSB_RELEASE_EXEC} -r --short 
                    OUTPUT_VARIABLE LINUX_VERSION 
                    OUTPUT_STRIP_TRAILING_WHITESPACE)
    message(STATUS "Linux Version: ${LINUX_VERSION}")
  endif()
endif()

# Detect CPU architecture
message(STATUS "CPU architecture: ${CMAKE_HOST_SYSTEM_PROCESSOR}")

if(FF_GPU_BACKEND STREQUAL "hip_cuda" OR FF_GPU_BACKEND STREQUAL "hip_rocm")
  set(ROCM_PATH "/opt/rocm" CACHE STRING "Default ROCM installation directory.")
endif()

# ZLIB
include(zlib)

# CUDA
if (FF_GPU_BACKEND STREQUAL "cuda" OR FF_GPU_BACKEND STREQUAL "hip_cuda")
  include(cuda)
endif()

# CUDNN
if (FF_GPU_BACKEND STREQUAL "cuda" OR FF_GPU_BACKEND STREQUAL "hip_cuda")
  include(cudnn)
endif()

# NCCL
if(FF_USE_NCCL)
  include(nccl)
  list(APPEND FF_CC_FLAGS
    -DFF_USE_NCCL)
  list(APPEND FF_NVCC_FLAGS
    -DFF_USE_NCCL)
endif()

# Legion
include(legion)

# json
include(json)

# variant
include(variant)

# optional
include(optional)

if(FF_USE_PYTHON)
  list(APPEND FF_CC_FLAGS
    -DBINDINGS_AUGMENT_PYTHONPATH)
  list(APPEND FF_NVCC_FLAGS
    -DBINDINGS_AUGMENT_PYTHONPATH)
endif()

if (FF_GPU_BACKEND STREQUAL "cuda")
  list(APPEND FF_CC_FLAGS
    -DFF_USE_CUDA)
  list(APPEND FF_NVCC_FLAGS
    -DFF_USE_CUDA)
elseif (FF_GPU_BACKEND STREQUAL "hip_cuda")
  list(APPEND FF_CC_FLAGS
    -DFF_USE_HIP_CUDA)
  list(APPEND FF_HIPCC_FLAGS
    -DFF_USE_HIP_CUDA)
elseif (FF_GPU_BACKEND STREQUAL "hip_rocm")
  list(APPEND FF_CC_FLAGS
    -DFF_USE_HIP_ROCM)
  list(APPEND FF_HIPCC_FLAGS
    -DFF_USE_HIP_ROCM)
else()
endif()

# Start build FlexFlow
if (CMAKE_BUILD_TYPE STREQUAL "Debug")
  list(APPEND FF_CC_FLAGS
    -DFF_DEBUG)
  list(APPEND FF_NVCC_FLAGS
    -DFF_DEBUG)
endif()

message(STATUS "FlexFlow MAX_DIM: ${FF_MAX_DIM}")

list(APPEND FF_CC_FLAGS
  -DMAX_TENSOR_DIM=${FF_MAX_DIM})

if(FF_USE_AVX2)
  list(APPEND FF_CC_FLAGS
    -DFF_USE_AVX2
    -mavx2)
endif()

list(APPEND FF_NVCC_FLAGS
  -Wno-deprecated-gpu-targets
  -DMAX_TENSOR_DIM=${FF_MAX_DIM})

list(APPEND FF_LD_FLAGS
  -lrt
  -ldl
  -rdynamic)

# Set FF FLAGS
add_compile_options(${FF_CC_FLAGS})
set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} ${FF_NVCC_FLAGS} -UNDEBUG)
link_libraries(${FF_LD_FLAGS})

list(APPEND FLEXFLOW_INCLUDE_DIRS
  ${FLEXFLOW_ROOT}/include
  ${FLEXFLOW_ROOT})

file(GLOB_RECURSE FLEXFLOW_HDR
  LIST_DIRECTORIES False
  ${FLEXFLOW_ROOT}/include/*.h)

file(GLOB_RECURSE FLEXFLOW_SRC
  LIST_DIRECTORIES False
  ${FLEXFLOW_ROOT}/src/*.cc)
list(REMOVE_ITEM FLEXFLOW_SRC "${FLEXFLOW_ROOT}/src/runtime/cpp_driver.cc")

set(FLEXFLOW_CPP_DRV_SRC
  ${FLEXFLOW_ROOT}/src/runtime/cpp_driver.cc)

add_library(substitution_loader SHARED
  ${FLEXFLOW_ROOT}/src/runtime/substitution_loader.cc)
target_include_directories(substitution_loader PRIVATE ${FLEXFLOW_INCLUDE_DIRS})
target_link_libraries(substitution_loader nlohmann_json::nlohmann_json)


#message("FLEXFLOW_INCLUDE_DIRS: ${FLEXFLOW_INCLUDE_DIRS}")

# compile flexflow lib
if (FF_GPU_BACKEND STREQUAL "cuda")
  file(GLOB_RECURSE FLEXFLOW_GPU_SRC
    LIST_DIRECTORIES False
    ${FLEXFLOW_ROOT}/src/*.cu)

  add_compile_definitions(FF_USE_CUDA)

  if(BUILD_SHARED_LIBS)
    cuda_add_library(flexflow SHARED ${FLEXFLOW_GPU_SRC} ${FLEXFLOW_SRC} OPTIONS ${CUDA_GENCODE})
  else()
    cuda_add_library(flexflow STATIC ${FLEXFLOW_GPU_SRC} ${FLEXFLOW_SRC} OPTIONS ${CUDA_GENCODE})
  endif()
elseif(FF_GPU_BACKEND STREQUAL "hip_cuda" OR FF_GPU_BACKEND STREQUAL "hip_rocm")
  file(GLOB_RECURSE FLEXFLOW_GPU_SRC
    LIST_DIRECTORIES False
    ${FLEXFLOW_ROOT}/src/*.cpp)

  if(BUILD_SHARED_LIBS)
    add_library(flexflow SHARED ${FLEXFLOW_GPU_SRC} ${FLEXFLOW_SRC})
  else()
    add_library(flexflow STATIC ${FLEXFLOW_GPU_SRC} ${FLEXFLOW_SRC})
  endif()

  list(APPEND CMAKE_PREFIX_PATH ${ROCM_PATH}/hip ${ROCM_PATH})

  find_package(hip REQUIRED)

  if (FF_GPU_BACKEND STREQUAL "hip_cuda")
    # The targets defined by the hip cmake config only target amd devices.
    # For targeting nvidia devices, we'll make our own interface target,
    # hip_device_nvidia, that includes the rocm and hip headers. 
    add_library(hip_device_nvidia INTERFACE)

    if (NOT FF_CUDA_ARCH STREQUAL "")
      target_compile_options(hip_device_nvidia INTERFACE -arch=compute_${FF_CUDA_ARCH})
    endif()

    target_include_directories(hip_device_nvidia SYSTEM INTERFACE ${HIP_INCLUDE_DIRS} ${ROCM_PATH}/include)
    target_include_directories(hip_device_nvidia INTERFACE ${HIP_INCLUDE_DIRS} ${ROCM_PATH}/include)

    add_compile_definitions(FF_USE_HIP_CUDA)

    # Linking cuda: 
    # We do not explicitly link cuda. hipcc when targeting nvidia will 
    # use nvcc under the hood. nvcc when used for linking will handle 
    # linking cuda dependencies
    target_link_libraries(flexflow hip_device_nvidia)
  elseif(FF_GPU_BACKEND STREQUAL "hip_rocm")
    find_package(hipblas REQUIRED)
    find_package(miopen REQUIRED)
    # find_package(rocrand REQUIRED)
    find_library(HIP_RAND_LIBRARY hiprand REQUIRED)

    add_compile_definitions(FF_USE_HIP_ROCM)

    # The hip cmake config module defines three targets, 
    # hip::amdhip64, hip::host, and hip::device.
    #
    # hip::host and hip::device are interface targets. hip::amdhip64 is an 
    # imported target for libamdhip.
    #
    # You do not directly link to hip::amdhip64. hip::host links to hip::amdhip64
    # and hip::device links to hip::host. Link to hip::host to just use hip without 
    # compiling any GPU code. Link to hip::device to compile the GPU device code.
    #
    # Docs (outdated):
    # https://rocmdocs.amd.com/en/latest/Installation_Guide/Using-CMake-with-AMD-ROCm.html
    target_link_libraries(flexflow hip::device roc::hipblas MIOpen ${HIP_RAND_LIBRARY})
  endif()
else()
  message(FATAL_ERROR "Unsupported FF_GPU_BACKEND for cmake: ${FF_GPU_BACKEND}")
endif()

target_include_directories(flexflow PUBLIC ${FLEXFLOW_INCLUDE_DIRS})
# LEGION_URL is defined if we found a precompiled Legion library to download
if(LEGION_URL)
  # Legion builds produce two library files: one for the Legion runtime and one for the Realm runtime. 
  # When linking FlexFlow to a precompiled version of Legion, we need to manually link to both library files.
  target_link_libraries(flexflow ${LEGION_LIBRARY} ${REALM_LIBRARY} nlohmann_json::nlohmann_json mpark_variant optional)
  add_dependencies(flexflow ${LEGION_NAME})
else()
  # When building Legion from source, we do so by calling add_subdirectory(), and obtain a library with both the
  # Legion and Realm runtimes. The library's name is saved into the LEGION_LIBRARY variable. Hence, we only need
  # to link FlexFlow to ${LEGION_LIBRARY}
  target_link_libraries(flexflow ${LEGION_LIBRARY} nlohmann_json::nlohmann_json mpark_variant optional)
endif()

if(FF_USE_NCCL)
  add_dependencies(flexflow ${NCCL_NAME})
endif()

# build binary
option(FF_BUILD_RESNET "build resnet example" OFF)
option(FF_BUILD_RESNEXT "build resnext example" OFF)
option(FF_BUILD_ALEXNET "build alexnet example" OFF)
option(FF_BUILD_DLRM "build DLRM example" OFF)
option(FF_BUILD_XDL "build XDL example" OFF)
option(FF_BUILD_INCEPTION "build inception example" OFF)
option(FF_BUILD_CANDLE_UNO "build candle uno example" OFF)
option(FF_BUILD_TRANSFORMER "build transformer example" OFF)
option(FF_BUILD_MOE "build mixture of experts example" OFF)
option(FF_BUILD_MLP_UNIFY "build mlp unify example" OFF)
option(FF_BUILD_SPLIT_TEST "build split test example" OFF)
option(FF_BUILD_SPLIT_TEST_2 "build split test 2 example" OFF)
option(FF_BUILD_ALL_EXAMPLES "build all examples. Overrides others" OFF)
option(FF_BUILD_UNIT_TESTS "build non-operator unit tests" OFF)
option(FF_BUILD_SUBSTITUTION_TOOL "build substitution conversion tool" OFF)
option(FF_BUILD_VISUALIZATION_TOOL "build substitution visualization tool" OFF)

if(FF_BUILD_UNIT_TESTS)
  set(BUILD_GMOCK OFF)
  add_subdirectory(deps/googletest)
  enable_testing()
  add_subdirectory(tests/unit)
endif()

if(FF_BUILD_SUBSTITUTION_TOOL)
  add_subdirectory(tools/protobuf_to_json)
endif()

if(FF_BUILD_VISUALIZATION_TOOL)
  add_subdirectory(tools/substitutions_to_dot)
endif()

# Python
if(FF_USE_PYTHON)
  add_subdirectory(deps/pybind11)
  add_subdirectory(python)
endif()

if(FF_BUILD_RESNET OR FF_BUILD_ALL_EXAMPLES)
  add_subdirectory(examples/cpp/ResNet)
endif()

if(FF_BUILD_RESNEXT OR FF_BUILD_ALL_EXAMPLES)
  add_subdirectory(examples/cpp/resnext50)
endif()

if(FF_BUILD_ALEXNET OR FF_BUILD_ALL_EXAMPLES)
  add_subdirectory(examples/cpp/AlexNet)
endif()

if(FF_BUILD_MLP_UNIFY OR FF_BUILD_ALL_EXAMPLES)
  add_subdirectory(examples/cpp/MLP_Unify)
endif()

if(FF_BUILD_SPLIT_TEST OR FF_BUILD_ALL_EXAMPLES)
  add_subdirectory(examples/cpp/split_test)
endif()

if(FF_BUILD_SPLIT_TEST_2 OR FF_BUILD_ALL_EXAMPLES)
  add_subdirectory(examples/cpp/split_test_2)
endif()

if(FF_BUILD_INCEPTION OR FF_BUILD_ALL_EXAMPLES)
  add_subdirectory(examples/cpp/InceptionV3)
endif()

#TODO: Once functional add to BUILD_ALL_EXAMPLES
if(FF_BUILD_CANDLE_UNO OR FF_BUILD_ALL_EXAMPLES)
  add_subdirectory(examples/cpp/candle_uno)
endif()

if(FF_BUILD_DLRM OR FF_BUILD_ALL_EXAMPLES)
  add_subdirectory(examples/cpp/DLRM)

  #add_executable(generate_dlrm_hetero_strategy src/runtime/dlrm_strategy_hetero.cc)
  #target_include_directories(generate_dlrm_hetero_strategy PUBLIC ${FLEXFLOW_INCLUDE_DIRS})

  #add_executable(generate_dlrm_strategy src/runtime/dlrm_strategy.cc)
  #target_include_directories(generate_dlrm_strategy PUBLIC ${FLEXFLOW_INCLUDE_DIRS})
endif()

if(FF_BUILD_XDL OR FF_BUILD_ALL_EXAMPLES)
  add_subdirectory(examples/cpp/XDL)
endif()

if(FF_BUILD_TRANSFORMER OR FF_BUILD_ALL_EXAMPLES)
  add_subdirectory(examples/cpp/Transformer)
endif()

if(FF_BUILD_MOE OR FF_BUILD_ALL_EXAMPLES)
  add_subdirectory(examples/cpp/mixture_of_experts)
endif()

# installation
set(INCLUDE_DEST "include")
set(LIB_DEST "lib")
install(FILES ${FLEXFLOW_HDR} DESTINATION ${INCLUDE_DEST})
install(TARGETS flexflow DESTINATION ${LIB_DEST})
