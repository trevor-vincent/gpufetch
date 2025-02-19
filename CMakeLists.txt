cmake_minimum_required(VERSION 3.10)
include(CheckLanguage)
include(ExternalProject)

project(gpufetch CXX)

set(SRC_DIR "src")
set(COMMON_DIR "${SRC_DIR}/common")
set(CUDA_DIR "${SRC_DIR}/cuda")
set(INTEL_DIR "${SRC_DIR}/intel")

if(NOT DEFINED ENABLE_INTEL_BACKEND)
    set(ENABLE_INTEL_BACKEND true)
endif()

if(NOT DEFINED ENABLE_CUDA_BACKEND OR ENABLE_CUDA_BACKEND)
  check_language(CUDA)
  if(CMAKE_CUDA_COMPILER)
    enable_language(CUDA)
    set(ENABLE_CUDA_BACKEND true)
    # Must link_directories early so add_executable(gpufetch ...) gets the right directories
    link_directories(cuda_backend ${CMAKE_CUDA_COMPILER_TOOLKIT_ROOT}/targets/x86_64-linux/lib)
  else()
    set(ENABLE_CUDA_BACKEND false)
  endif()
endif()

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")
find_package(PCIUTILS)
if(NOT ${PCIUTILS_FOUND})
  message(STATUS "${BoldYellow}pciutils not found, downloading and building a local copy...${ColorReset}")

  # Download and build pciutils
  set(PCIUTILS_INSTALL_LOCATION ${CMAKE_BINARY_DIR}/pciutils-install)
  ExternalProject_Add(pciutils
    GIT_REPOSITORY https://github.com/pciutils/pciutils
    CONFIGURE_COMMAND ""
    BUILD_COMMAND make SHARED=no HWDB=no
    BUILD_IN_SOURCE true
    INSTALL_COMMAND make PREFIX=${PCIUTILS_INSTALL_LOCATION} install-lib
  )

  include_directories(${PCIUTILS_INSTALL_LOCATION}/include)
  link_directories(${PCIUTILS_INSTALL_LOCATION}/lib)
else()
  include_directories(${PCIUTILS_INCLUDE_DIR})
  link_libraries(${PCIUTILS_LIBRARIES})
endif()

add_executable(gpufetch ${COMMON_DIR}/main.cpp ${COMMON_DIR}/args.cpp ${COMMON_DIR}/gpu.cpp ${COMMON_DIR}/pci.cpp ${COMMON_DIR}/sort.cpp ${COMMON_DIR}/global.cpp ${COMMON_DIR}/printer.cpp ${COMMON_DIR}/master.cpp ${COMMON_DIR}/uarch.cpp)
set(SANITY_FLAGS "-Wfloat-equal -Wshadow -Wpointer-arith")
set(CMAKE_CXX_FLAGS "${SANITY_FLAGS} -Wall -Wextra -pedantic -fstack-protector-all -pedantic -std=c++11")

if(ENABLE_INTEL_BACKEND)
  target_compile_definitions(gpufetch PUBLIC BACKEND_INTEL)

  add_library(intel_backend STATIC ${INTEL_DIR}/intel.cpp ${INTEL_DIR}/pci.cpp ${INTEL_DIR}/uarch.cpp ${INTEL_DIR}/udev.cpp ${INTEL_DIR}/cpuid.cpp)

  if(NOT ${PCIUTILS_FOUND})
    add_dependencies(intel_backend pciutils)
  endif()

  target_link_libraries(gpufetch intel_backend)
endif()

if(ENABLE_CUDA_BACKEND)
  target_compile_definitions(gpufetch PUBLIC BACKEND_CUDA)

  # https://en.wikipedia.org/w/index.php?title=CUDA&section=5#GPUs_supported
  # https://raw.githubusercontent.com/PointCloudLibrary/pcl/master/cmake/pcl_find_cuda.cmake
  if(${CMAKE_CUDA_COMPILER_VERSION} VERSION_GREATER_EQUAL "11.1")
    set(CMAKE_CUDA_ARCHITECTURES 35 37 50 52 53 60 61 62 70 72 75 80 86)
  elseif(${CMAKE_CUDA_COMPILER_VERSION} EQUAL "11.0")
    set(CMAKE_CUDA_ARCHITECTURES 30 32 35 37 50 52 53 60 61 62 70 72 75 80)
  elseif(${CMAKE_CUDA_COMPILER_VERSION} VERSION_GREATER_EQUAL "10.0")
    set(CMAKE_CUDA_ARCHITECTURES 30 32 35 37 50 52 53 60 61 62 70 72 75)
  elseif(${CMAKE_CUDA_COMPILER_VERSION} VERSION_GREATER_EQUAL "9.0")
    set(CMAKE_CUDA_ARCHITECTURES 30 32 35 37 50 52 53 60 61 62 70 72)
  elseif(${CMAKE_CUDA_COMPILER_VERSION} VERSION_EQUAL "8.0")
    set(CMAKE_CUDA_ARCHITECTURES 20 21 30 32 35 37 50 52 53 60 61 62)
  endif()

  # https://docs.nvidia.com/cuda/cuda-samples/index.html#new-features-in-cuda-toolkit-11-6
  # Not sure about this. Why the heck did they change this?
  if(${CMAKE_CUDA_COMPILER_VERSION} VERSION_GREATER_EQUAL "11.6")
    include(FetchContent)
    FetchContent_Declare(
      cudasamples
      GIT_REPOSITORY https://github.com/NVIDIA/cuda-samples
    )
    FetchContent_MakeAvailable(cudasamples)
    set(CUDA_SAMPLES_PATH ${cudasamples_SOURCE_DIR}/Common)
  else()
    set(CUDA_SAMPLES_PATH ${CMAKE_CUDA_COMPILER_TOOLKIT_ROOT}/samples/common/inc)
  endif()

  add_library(cuda_backend STATIC ${CUDA_DIR}/cuda.cpp ${CUDA_DIR}/uarch.cpp ${CUDA_DIR}/pci.cpp)

  if(NOT ${PCIUTILS_FOUND})
    add_dependencies(cuda_backend pciutils)
  endif()

  target_include_directories(cuda_backend PUBLIC ${CUDA_SAMPLES_PATH} ${CMAKE_CUDA_COMPILER_TOOLKIT_ROOT}/targets/x86_64-linux/include)

  target_link_libraries(cuda_backend PRIVATE cudart)
  target_link_libraries(gpufetch cuda_backend)
endif()

target_link_libraries(gpufetch pci z)
install(TARGETS gpufetch DESTINATION bin)

if(NOT WIN32)
  string(ASCII 27 Esc)
  set(ColorReset "${Esc}[m")
  set(ColorBold  "${Esc}[1m")
  set(Red         "${Esc}[31m")
  set(Green       "${Esc}[32m")
  set(BoldRed     "${Esc}[1;31m")
  set(BoldGreen   "${Esc}[1;32m")
  set(BoldYellow  "${Esc}[1;33m")
endif()

message(STATUS "----------------------")
message(STATUS "gpufetch build report:")
if(ENABLE_CUDA_BACKEND)
  message(STATUS "CUDA backend: ${BoldGreen}ON${ColorReset}")
else()
  message(STATUS "CUDA backend: ${BoldRed}OFF${ColorReset}")
endif()
if(ENABLE_INTEL_BACKEND)
  message(STATUS "Intel backend: ${BoldGreen}ON${ColorReset}")
else()
  message(STATUS "Intel backend: ${BoldRed}OFF${ColorReset}")
endif()
message(STATUS "----------------------")
