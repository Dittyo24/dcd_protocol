# ##################################################################################################
# Defines the dcd-vm library and associated tooling. See docs/cmake.md for instructions on how to
# build dcd-vm or integrate with another system with CMake.
# ##################################################################################################
cmake_minimum_required(VERSION 3.8)
set(VERSION_MAJOR 1)
set(VERSION_MINOR 0)
set(VERSION_PATCH 0)
set(VERSION_SUFFIX rc1)
project(dcd-vm VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH})

if (VERSION_SUFFIX)
   set(VERSION_FULL "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}-${VERSION_SUFFIX}")
else()
   set(VERSION_FULL "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}")
endif()

message(STATUS "Building dcd-vm v${VERSION_FULL}...")

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_EXTENSIONS ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(CMAKE_CXX_FLAGS "-fdiagnostics-color=always")
set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
find_package(Threads)
link_libraries(Threads::Threads)

# ##################################################################################################
# Gate against Windows builds.
# ##################################################################################################
if (WIN32)
   message(FATAL_ERROR "DCD-VM currently only builds on Unix like OSes (Linux, Mac OSX)")
endif()

# ##################################################################################################
# Defined options for building or integrating dcd-vm.
# ##################################################################################################
include(CMakeDependentOption)
option(ENABLE_SOFTFLOAT "enable the backend to use deterministic software floating point operations"
	ON)
option(FULL_DEBUG_BUILD "enables stack dumping and instruction tracing" OFF)
option(ENABLE_INSTALL "enable this library to be installed" ON)
option(ENABLE_MEMORY_OPS_ALIGNMENT "enable the backend to obey alignment hints" OFF)
option(ENABLE_TOOLS "enable building of tools" ON)
option(ENABLE_TESTS "enable building of unit tests, spec tests" OFF)
cmake_dependent_option(ENABLE_SPEC_TESTS "enable wasm spec tests" ON "ENABLE_TESTS" ON)
cmake_dependent_option(ENABLE_FUZZ_TESTS "enable fuzz testing" OFF "ENABLE_TESTS" ON)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/modules")
include(DcdVMBuildUtils)

# ##################################################################################################
# Create the dcd-vm library.
# ##################################################################################################
add_library(dcd-vm INTERFACE)
target_include_directories(dcd-vm
                           INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include
                                     ${CMAKE_CURRENT_SOURCE_DIR}/external/softfloat/source/include)
# ignore the C++17 register warning until clean up
target_compile_options( dcd-vm INTERFACE "-Wno-register" )

# ##################################################################################################
# Enable debugging stats for dcd-vm.
# ##################################################################################################
if(FULL_DEBUG_BUILD)
   target_compile_definitions(dcd-vm INTERFACE -DDCD_VM_FULL_DEBUG)
endif()

# ##################################################################################################
# Option specific build configurations.
# ##################################################################################################
if(ENABLE_SOFTFLOAT)
   if(NOT TARGET softfloat)
      add_subdirectory(external)
   endif()
   target_compile_definitions(dcd-vm INTERFACE -DDCD_VM_SOFTFLOAT)
   target_link_libraries(dcd-vm INTERFACE softfloat)
endif()

if(ENABLE_MEMORY_OPS_ALIGNMENT)
   target_compile_definitions(dcd-vm INTERFACE -DDCD_VM_ALIGN_MEMORY_OPS)
endif()

# ##################################################################################################
# Build dcd-vm tools.
# ##################################################################################################
if(ENABLE_TOOLS)
add_subdirectory(tools)
endif()

# ##################################################################################################
# Build dcd-vm tests.
# ##################################################################################################
if(ENABLE_TESTS)
   include(CTest)
   set(CATCH_BUILD_TESTING OFF CACHE BOOL "disable building tests")
   set(CATCH_INSTALL_DOCS OFF CACHE BOOL "disable installing docs")
   set(CATCH_INSTALL_HELPERS OFF CACHE BOOL "disable installing helpers")
   add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/external/Catch2)
   include(external/Catch2/contrib/Catch.cmake)
   add_subdirectory(tests)
   if (NOT EXISTS ${CMAKE_BINARY_DIR}/wasms AND ENABLE_SPEC_TESTS)
      include(ExternalProject)
      ExternalProject_Add(
         dcd_vm_test_wasms
         GIT_REPOSITORY "https://github.com/DCD/dcd-vm-test-wasms.git"
         SOURCE_DIR ${CMAKE_BINARY_DIR}/wasms
         BINARY_DIR ${CMAKE_BINARY_DIR}/wasms
         BUILD_ALWAYS 0
         BUILD_COMMAND ""
         TEST_COMMAND ""
         INSTALL_COMMAND ""
         UPDATE_COMMAND ""
         PATCH_COMMAND ""
         CONFIGURE_COMMAND ""
      )
   endif()
endif()

# ##################################################################################################
# Installation.
# ##################################################################################################
if(ENABLE_INSTALL)
   include(GNUInstallDirs)
   message(STATUS "Installing...")
   install(TARGETS dcd-vm
            LIBRARY
               DESTINATION ${CMAKE_INSTALL_LIBDIR}
               COMPONENT Libraries
            PUBLIC_HEADER
               DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
               COMPONENT Headers)

   install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/dcd
      DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
endif()

