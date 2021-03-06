#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#
cmake_minimum_required (VERSION 2.8.12)

enable_language(CXX)

find_package(ProtonCpp REQUIRED)
set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
find_package(Threads)

include_directories(${ProtonCpp_INCLUDE_DIRS})
link_libraries(${ProtonCpp_LIBRARIES})
add_definitions(${ProtonCpp_DEFINITIONS})


macro (has_cxx_features result)
set(${result} OFF)
if (DEFINED CMAKE_CXX_COMPILE_FEATURES)
  set(${result} ON)
  foreach(feature ${ARGN})
    list(FIND CMAKE_CXX_COMPILE_FEATURES cxx_${feature} N)
    if (N EQUAL -1)
      set(${result} OFF)
      break()
    endif()
  endforeach()
endif()
endmacro()

set (BUILD_CPP_03 OFF CACHE BOOL "Compile as C++03 even when C++11 is available")
# This effectively checks for cmake version 3.1 or later
if (DEFINED CMAKE_CXX_COMPILE_FEATURES)
  if (BUILD_CPP_03)
    set(STD 98)
  else ()
    set(STD 11)
    has_cxx_features(HAS_ENOUGH_CPP11 lambdas variadic_templates)
    message(STATUS "Compiling C++11 examples: ${HAS_ENOUGH_CPP11}")
  endif ()
  set(CMAKE_CXX_STANDARD ${STD})
  set(CMAKE_CXX_EXTENSIONS OFF)
endif()

# Single-threaded examples that work on C++03
foreach(example
    broker
    helloworld
    simple_recv
    simple_send
    reconnect_client
    message_properties
    scheduled_send_03
    direct_recv
    direct_send
    client
    server
    server_direct
    connection_options
    queue_browser
    selected_recv
    flow_control
    ssl
    ssl_client_cert
    encode_decode)
  add_executable(${example} ${example}.cpp)
endforeach()

if(HAS_ENOUGH_CPP11)
  # Examples that require C++11
  foreach(example
      scheduled_send
      service_bus)
    add_executable(${example} ${example}.cpp)
  endforeach()

  # Examples that use threads directly
  if (Threads_FOUND)
    foreach(example
        multithreaded_client
        multithreaded_client_flow_control)
      add_executable(${example} ${example}.cpp)
      target_link_libraries(${example} ${CMAKE_THREAD_LIBS_INIT})
    endforeach()
  endif()
endif()

# Add a test with the correct environment to find test executables and valgrind.
macro(add_cpp_test name)
  if(WIN32)
    set(test_path "$<TARGET_FILE_DIR:broker>;$<TARGET_FILE_DIR:qpid-proton>;$<TARGET_FILE_DIR:qpid-proton-cpp>")
  else(WIN32)
    set(test_path "$<TARGET_FILE_DIR:broker>:$ENV{PATH}")
  endif(WIN32)
  set(run_env ${PYTHON_EXECUTABLE} ${CMAKE_SOURCE_DIR}/proton-c/env.py ${EXAMPLE_ENV})
  add_test(NAME ${name} COMMAND ${run_env} "PATH=${test_path}" ${VALGRIND_ENV} "HAS_CPP11=$<$<BOOL:${HAS_ENOUGH_CPP11}>:1>" -- ${ARGN})
endmacro()

add_cpp_test(cpp-example-container ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/example_test.py -v ContainerExampleTest)

if (NOT SSL_IMPL STREQUAL none)
add_cpp_test(cpp-example-container-ssl ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/example_test.py -v ContainerExampleSSLTest)
endif()
