# Copyright (c) Facebook, Inc. and its affiliates.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
cmake_minimum_required(VERSION 3.10)
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/CMake" ${CMAKE_MODULE_PATH})

# set the project name
project(velox)

# If CODEGEN support isn't explicitly set, we guestimate the value based on the
# compiler
if((NOT DEFINED CODEGEN_SUPPORT) AND (CMAKE_CXX_COMPILER_ID MATCHES "Clang"))
  message(STATUS "Enabling Codegen")
  set(CODEGEN_SUPPORT True)
else()
  message(STATUS "Disabling Codegen")
  set(CODEGEN_SUPPORT False)
endif()

# define processor variable for conditional compilation
if(${CODEGEN_SUPPORT})
  add_compile_definitions(CODEGEN_ENABLED=1)
endif()

# MacOSX enables two-level namespace by default:
# http://mirror.informatimago.com/next/developer.apple.com/releasenotes/DeveloperTools/TwoLevelNamespaces.html
# Enables -flat_namespace so type_info can be deudplicated across .so boundaries
if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
  add_link_options("-Wl,-flat_namespace")
endif()

if(UNIX AND NOT APPLE)
  # codegen linker flags, -export-dynamic for rtti
  add_link_options("-Wl,-export-dynamic")
endif()

# Required so velox code can be used in a dynamic library
set(POSITION_INDEPENDENT_CODE True)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED True)
set(CMAKE_CXX_FLAGS
    "${CMAKE_CXX_FLAGS} -mavx2 -mfma -mavx -mf16c  -mbmi2 -march=native -D USE_VELOX_COMMON_BASE -g"
)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D HAS_UNCAUGHT_EXCEPTIONS")

# Under Ninja, we are able to designate certain targets large enough to require
# restricted parallelism.
if("${MAX_HIGH_MEM_JOBS}")
  set_property(GLOBAL PROPERTY JOB_POOLS
                               "high_memory_pool=${MAX_HIGH_MEM_JOBS}")
else()
  set_property(GLOBAL PROPERTY JOB_POOLS high_memory_pool=1000)
endif()

if("${MAX_LINK_JOBS}")
  set_property(GLOBAL APPEND PROPERTY JOB_POOLS
                                      "link_job_pool=${MAX_LINK_JOBS}")
  set(CMAKE_JOB_POOL_LINK link_job_pool)
endif()

if("${TREAT_WARNINGS_AS_ERRORS}")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror")
endif()

if("${ENABLE_ALL_WARNINGS}")
  if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(KNOWN_COMPILER_SPECIFIC_WARNINGS
        "-Wno-range-loop-analysis -Wno-mismatched-tags")
  elseif(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    set(KNOWN_COMPILER_SPECIFIC_WARNINGS
        "-Wno-implicit-fallthrough \
         -Wno-empty-body \
         -Wno-class-memaccess \
         -Wno-comment \
         -Wno-int-in-bool-context \
         -Wno-redundant-move \
         -Wno-type-limits")
  endif()

  set(KNOWN_WARNINGS
      "-Wno-unused \
       -Wno-unused-parameter \
       -Wno-sign-compare \
       -Wno-ignored-qualifiers \
       ${KNOWN_COMPILER_SPECIFIC_WARNINGS}")

  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra ${KNOWN_WARNINGS}")
endif()

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(Boost_USE_MULTITHREADED TRUE)
find_package(
  Boost
  1.66.0
  REQUIRED
  program_options
  context
  filesystem
  regex
  thread
  system
  date_time
  atomic)
include_directories(SYSTEM ${Boost_INCLUDE_DIRS})

# Range-v3 will be enable when the codegen code actually lands keeping it here
# for reference. find_package(range-v3)

find_package(GTest REQUIRED)
find_package(gflags COMPONENTS shared)

find_library(GMock gmock)

find_library(GLOG glog)
include_directories(SYSTEM ${GTEST_INCLUDE_DIRS})

find_library(FMT fmt)

find_library(EVENT event)

find_library(DOUBLE_CONVERSION double-conversion)

find_library(LZ4 lz4)
find_library(LZO lzo2)
find_library(RE2 re2 REQUIRED)
find_library(ZSTD zstd)
find_package(ZLIB)
find_library(SNAPPY snappy)

if(CMAKE_SYSTEM_NAME MATCHES "Darwin")
  set(CMAKE_PREFIX_PATH "/usr/local/opt/icu4c" ${CMAKE_PREFIX_PATH})
  find_package(ICU REQUIRED)
  include_directories(${ICU_INCLUDE_DIRS})
  link_directories("${ICU_INCLUDE_DIRS}/../lib")
endif()

find_package(folly CONFIG REQUIRED)
set(FOLLY_WITH_DEPENDENCIES
    ${FOLLY_LIBRARIES} ${Boost_LIBRARIES} ${DOUBLE_CONVERSION_LIBRARIES}
    ${EVENT} ${SNAPPY} ${CMAKE_DL_LIBS})

set(FOLLY ${FOLLY_LIBRARIES})
set(FOLLY_BENCHMARK Folly::follybenchmark)

find_package(BZip2 MODULE)
if(BZIP2_FOUND)
  list(APPEND FOLLY_WITH_DEPENDENCIES ${BZIP2_LIBRARIES})
endif()

include_directories(SYSTEM ${FOLLY_INCLUDE_DIRS})

find_package(Protobuf REQUIRED)
include_directories(SYSTEM ${Protobuf_INCLUDE_DIRS})

# GCC needs to link a library to enable std::filesystem.
if("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU")
  set(FILESYSTEM "stdc++fs")

  # Ensure we have gcc at least 8+.
  if(CMAKE_CXX_COMPILER_VERSION LESS 8.0)
    message(
      FATAL_ERROR "VELOX requires gcc > 8. Found ${CMAKE_CXX_COMPILER_VERSION}")
  endif()
else()
  set(FILESYSTEM "")
endif()

include_directories(SYSTEM velox)
include_directories(SYSTEM velox/external)
include_directories(SYSTEM velox/external/duckdb)

include(CTest) # include after project() but before add_subdirectory()

include_directories(.)

# TODO: Include all other installation files. For now just making sure this
# generates an installable makefile.
install(FILES velox/type/Type.h DESTINATION "include/velox")

add_subdirectory(velox)
