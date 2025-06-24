# Copyright 2024 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_minimum_required(VERSION 3.19)
project(fuzztest)

option(FUZZTEST_BUILD_TESTING "Building the tests." OFF)
option(FUZZTEST_FUZZING_MODE "Building the fuzztest in fuzzing mode." OFF)
set(FUZZTEST_COMPATIBILITY_MODE "" CACHE STRING "Compatibility mode. Available options: <empty>, libfuzzer")
set(CMAKE_CXX_STANDARD 17)

if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  set (COMPILER_GCC 1)
elseif (CMAKE_CXX_COMPILER_ID MATCHES "AppleClang")
  set (COMPILER_CLANG 1) # Safe to treat AppleClang as a regular Clang, in general.
elseif (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  set (COMPILER_CLANG 1)
else ()
  message (FATAL_ERROR "Compiler ${CMAKE_CXX_COMPILER_ID} is not supported")
endif ()

if (COMPILER_GCC AND (FUZZTEST_FUZZING_MODE OR (FUZZTEST_COMPATIBILITY_MODE STREQUAL "libfuzzer")))
  message (FATAL_ERROR "Compilation with GCC is not yet supported for fuzztest mode. Please use Clang. CC=clang CXX=clang++")
endif ()

if (FUZZTEST_FUZZING_MODE AND NOT FUZZTEST_COMPATIBILITY_MODE STREQUAL "")
  message (FATAL_ERROR "Please either use fuzzing mode or compatibility mode")
endif ()

if (NOT FUZZTEST_COMPATIBILITY_MODE STREQUAL "" AND
    NOT FUZZTEST_COMPATIBILITY_MODE STREQUAL "libfuzzer")
  message (FATAL_ERROR "Compatibility mode is only supported for libfuzzer")
endif ()

include(cmake/AddFuzzTest.cmake)
include(cmake/FuzzTestFlagSetup.cmake)

fuzztest_setup_fuzzing_flags()

include(cmake/BuildDependencies.cmake)
include(cmake/FuzzTestHelpers.cmake)
include(cmake/CompatibilityModeLinkLibFuzzer.cmake)

if (FUZZTEST_BUILD_TESTING)
  enable_testing()

  set(protobuf_PROTOC_EXE "${protobuf_BINARY_DIR}/protoc")
  include(${protobuf_SOURCE_DIR}/cmake/protobuf-generate.cmake)
endif ()

add_subdirectory(common)
add_subdirectory(fuzztest)
add_subdirectory(fuzztest/grammars)
add_subdirectory(fuzztest/internal)
add_subdirectory(fuzztest/internal/domains)
add_subdirectory(grammar_codegen)
add_subdirectory(grammar_codegen/generated_antlr_parser)
add_subdirectory(tools)

if (FUZZTEST_BUILD_TESTING)
  add_subdirectory(domain_tests)
  add_subdirectory(e2e_tests)
  add_subdirectory(e2e_tests/testdata)
endif ()

# ---------------------------------------------------------------------------
# FuzzTest installation and CMake package config for find_package support
# ---------------------------------------------------------------------------
include(GNUInstallDirs)

# List all top-level public targets you want to export (add/remove as needed)
set(FUZZTEST_EXPORT_TARGETS
    fuzztest
    fuzztest_core
    fuzztest_gtest_main
    fuzztest_macros
    googletest_fixture_adapter
    init_fuzztest
    llvm_fuzzer_main
    llvm_fuzzer_wrapper
)

install(TARGETS ${FUZZTEST_EXPORT_TARGETS}
    EXPORT FuzzTestTargets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

install(DIRECTORY fuzztest/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/fuzztest
    FILES_MATCHING PATTERN "*.h"
)

include(CMakePackageConfigHelpers)
configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/FuzzTestConfig.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/FuzzTestConfig.cmake"
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/FuzzTest
)

install(EXPORT FuzzTestTargets
    FILE FuzzTestTargets.cmake
    NAMESPACE fuzztest::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/FuzzTest
)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/FuzzTestConfig.cmake"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/FuzzTest
)
