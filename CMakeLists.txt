cmake_minimum_required(VERSION 3.5)
project(epigraph)

set(CMAKE_CXX_STANDARD 17)

add_library(epigraph SHARED)
target_include_directories(epigraph PUBLIC include solvers/wrappers/include)
target_include_directories(epigraph SYSTEM PUBLIC solvers)
target_sources(epigraph PRIVATE
    src/parameter.cpp
    src/variable.cpp
    src/expressions.cpp
    src/constraint.cpp
    src/problem.cpp

    solvers/wrappers/src/wrapperBase.cpp

    solvers/wrappers/src/socpWrapperBase.cpp
    solvers/wrappers/src/qpWrapperBase.cpp
)

# ==== Solvers ====
# Make sure the repository is cloned with --recurse-submodules.

# Note that enabling certain solvers automatically alters the license.
# Set ENABLE_OSQP to TRUE to enable OSQP solver support. Apache-2.0 License.
# Set ENABLE_ECOS to TRUE to enable ECOS solver support. GPLv3 License.
# Set ENABLE_EICOS to TRUE to enable EiCOS solver support. MIT License.

if(NOT ENABLE_OSQP AND NOT ENABLE_ECOS AND NOT ENABLE_EICOS)
  message(FATAL_ERROR "Epigraph: No solvers enabled. Set the appropriate variables for the desired solvers and re-run cmake.")
endif()

# OSQP
if(ENABLE_OSQP)
  target_sources(epigraph PRIVATE solvers/wrappers/src/osqpWrapper.cpp)
  target_compile_definitions(epigraph PUBLIC ENABLE_OSQP)
  add_subdirectory(solvers/osqp)
  target_link_libraries(epigraph osqp)
  message("Epigraph: OSQP enabled.")
else()
  message("Epigraph: OSQP disabled.")
endif()

# ECOS
if(ENABLE_ECOS)
  target_sources(epigraph PRIVATE solvers/wrappers/src/ecosWrapper.cpp)
  target_compile_definitions(epigraph PUBLIC ENABLE_ECOS)
  execute_process(COMMAND make shared
                  WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}/solvers/ecos)
  target_include_directories(epigraph SYSTEM PUBLIC solvers/ecos/external/SuiteSparse_config)
  target_link_libraries(epigraph
        ${CMAKE_CURRENT_LIST_DIR}/solvers/ecos/libecos.so)
  message("Epigraph: ECOS enabled.")
else()
  message("Epigraph: ECOS disabled.")
endif()

# EiCOS
if(ENABLE_EICOS)
  target_sources(epigraph PRIVATE solvers/wrappers/src/eicosWrapper.cpp)
  target_compile_definitions(epigraph PUBLIC ENABLE_EICOS)
  add_subdirectory(solvers/eicos)
  target_link_libraries(epigraph eicos)
  message("Epigraph: EiCOS enabled.")
else()
  message("Epigraph: EiCOS disabled.")
endif()

set(DEBUG_OPTIONS -Wall -Wextra -Wpedantic)
set(RELEASE_OPTIONS -O2)

find_package(Eigen3 REQUIRED NO_MODULE)
target_link_libraries(epigraph Eigen3::Eigen)

# ==== Testing ====
find_package(Catch2)
option(ENABLE_TESTING "Build tests. Requires Catch2 and fmt to be installed."
       FALSE)
if(Catch2_FOUND AND ENABLE_TESTING)
  if(NOT
     (ENABLE_OSQP
      AND ENABLE_ECOS
      AND ENABLE_EICOS))
    message(FATAL_ERROR "Epigraph: Enable all solvers to enable testing.")
  endif()

  include_directories(epigraph tests)
  add_executable(run_tests tests/run_tests.cpp)
  target_link_libraries(run_tests epigraph)
  message("Epigraph: Testing is enabled.")

  # fmt
  find_package(fmt REQUIRED)
  if(${fmt_FOUND})
    message(STATUS "Epigraph: Found fmt.")
    target_link_libraries(run_tests fmt::fmt)
  endif()

  # Coverage
  if(CMAKE_COMPILER_IS_GNUCXX)
    option(ENABLE_COVERAGE "Enable coverage reporting for gcc/clang" FALSE)
    if(ENABLE_COVERAGE)
      message("Epigraph: Code coverage is enabled.")
      list(APPEND DEBUG_OPTIONS --coverage -O0)
      target_link_libraries(epigraph --coverage)
    endif()
  endif()
endif()
# ====

target_compile_options(epigraph PUBLIC "$<$<CONFIG:DEBUG>:${DEBUG_OPTIONS}>")
target_compile_options(epigraph PUBLIC "$<$<CONFIG:RELEASE>:${RELEASE_OPTIONS}>")
