# Copyright 2010-2020, Olivier Stasse, Guilhem Saurel, JRL, CNRS/AIST, LAAS-CNRS

ADD_DEFINITIONS(-DBOOST_TEST_DYN_LINK -DBOOST_TEST_MAIN)

ADD_DEFINITIONS(-DTESTS_DATADIR="${CMAKE_CURRENT_SOURCE_DIR}/data")
ADD_DEFINITIONS(-DTESTS_PLUGINDIR="${LIBRARY_OUTPUT_PATH}")
ADD_DEFINITIONS(-DTESTS_DYNLIBSUFFIX="${CMAKE_SHARED_LIBRARY_SUFFIX}")

IF(DEFINED DISABLE_TESTS)
  MESSAGE(AUTHOR_WARNING "DISABLE_TESTS is deprecated. Use BUILD_TESTING instead.")
  IF(DISABLE_TESTS)
    SET(BUILD_TESTING OFF CACHE BOOL "")
  ELSE()
    SET(BUILD_TESTING ON CACHE BOOL "")
  ENDIF()
ENDIF(DEFINED DISABLE_TESTS)

IF(NOT TARGET build_tests)
  ADD_CUSTOM_TARGET(build_tests)
ENDIF()

IF(NOT DEFINED ctest_build_tests_exists)
  SET_PROPERTY(GLOBAL PROPERTY ctest_build_tests_exists OFF)
ENDIF(NOT DEFINED ctest_build_tests_exists)

#.rst:
# .. command:: CREATE_CTEST_BUILD_TESTS_TARGET
#
#    Create target ctest_build_tests if does not exist yet.
#
MACRO(CREATE_CTEST_BUILD_TESTS_TARGET)
  GET_PROPERTY(ctest_build_tests_exists_value GLOBAL PROPERTY ctest_build_tests_exists)
  IF(NOT BUILD_TESTING)
    IF(NOT ctest_build_tests_exists_value)
      ADD_TEST(ctest_build_tests "${CMAKE_COMMAND}" --build ${CMAKE_BINARY_DIR} --target build_tests -- $ENV{MAKEFLAGS})
      SET_PROPERTY(GLOBAL PROPERTY ctest_build_tests_exists ON)
    ENDIF(NOT ctest_build_tests_exists_value)
  ENDIF(NOT BUILD_TESTING)
ENDMACRO(CREATE_CTEST_BUILD_TESTS_TARGET)

#.rst:
# .. command:: ADD_UNIT_TEST (NAME SOURCE)
#
#   The behaviour of this function depends on :variable:`BUILD_TESTING` option.
#
MACRO(ADD_UNIT_TEST NAME SOURCE)
  CREATE_CTEST_BUILD_TESTS_TARGET()

  IF(NOT BUILD_TESTING)
    ADD_EXECUTABLE(${NAME} EXCLUDE_FROM_ALL ${SOURCE})
  ELSE(NOT BUILD_TESTING)
    ADD_EXECUTABLE(${NAME} ${SOURCE})
  ENDIF(NOT BUILD_TESTING)

  ADD_DEPENDENCIES(build_tests ${NAME})

  ADD_TEST(${NAME} ${RUNTIME_OUTPUT_DIRECTORY}/${NAME})
  # Support definition of DYLD_LIBRARY_PATH for OSX systems
  IF(APPLE)
    SET_TESTS_PROPERTIES(${NAME} PROPERTIES ENVIRONMENT "DYLD_LIBRARY_PATH=$ENV{DYLD_LIBRARY_PATH}")
    SET_TESTS_PROPERTIES(${NAME} PROPERTIES ENVIRONMENT "LD_LIBRARY_PATH=$ENV{LD_LIBRARY_PATH}")
  ENDIF(APPLE)

  IF(NOT BUILD_TESTING)
    SET_TESTS_PROPERTIES(${NAME} PROPERTIES DEPENDS ctest_build_tests)
  ENDIF(NOT BUILD_TESTING)
ENDMACRO(ADD_UNIT_TEST NAME SOURCE)

#.rst:
# .. command:: ADD_PYTHON_UNIT_TEST (NAME SOURCE [MODULES...])
#
#   Add a test called `NAME` that runs an equivalent of ``python ${SOURCE}``,
#   optionnaly with a `PYTHONPATH` set to `CMAKE_BINARY_DIR/MODULE_PATH` for each MODULES
#   `SOURCE` is relative to `PROJECT_SOURCE_DIR`
#
#   .. note:: :command:`FINDPYTHON` should have been called first.
#
MACRO(ADD_PYTHON_UNIT_TEST NAME SOURCE)
  ADD_TEST(NAME ${NAME} COMMAND ${PYTHON_EXECUTABLE} "${PROJECT_SOURCE_DIR}/${SOURCE}")
  SET(PYTHONPATH)

  SET(MODULES "${ARGN}")  # ARGN is not a variable
  FOREACH(MODULE_PATH IN LISTS MODULES)
    LIST(APPEND PYTHONPATH "${CMAKE_BINARY_DIR}/${MODULE_PATH}")
  ENDFOREACH(MODULE_PATH IN LISTS MODULES)

  IF(DEFINED ENV{PYTHONPATH})
    LIST(APPEND PYTHONPATH "$ENV{PYTHONPATH}")
  ENDIF(DEFINED ENV{PYTHONPATH})

  # get path separator to join those paths
  EXECUTE_PROCESS(COMMAND
      "${PYTHON_EXECUTABLE}" "-c" "import os; print(os.pathsep)"
      OUTPUT_VARIABLE PATHSEP
      OUTPUT_STRIP_TRAILING_WHITESPACE)

  STRING(REPLACE ";" "${PATHSEP}" PYTHONPATH_STR "${PYTHONPATH}")
  SET(ENV_VARIABLES "PYTHONPATH=${PYTHONPATH_STR}")
  IF(APPLE)
    LIST(APPEND ENV_VARIABLES "LD_LIBRARY_PATH=$ENV{LD_LIBRARY_PATH}")
    LIST(APPEND ENV_VARIABLES "DYLD_LIBRARY_PATH=$ENV{DYLD_LIBRARY_PATH}")
  ENDIF(APPLE)
  SET_TESTS_PROPERTIES(${NAME} PROPERTIES ENVIRONMENT "${ENV_VARIABLES}")
ENDMACRO(ADD_PYTHON_UNIT_TEST NAME SOURCE)

# DEFINE_UNIT_TEST(NAME LIB)
# ----------------------
#
# Compile a program and add it as a test
#
MACRO(DEFINE_UNIT_TEST NAME LIB)
  ADD_UNIT_TEST(${NAME} ${NAME}.cc)
  TARGET_INCLUDE_DIRECTORIES(${NAME} SYSTEM PUBLIC ${EIGEN3_INCLUDE_DIR})
  TARGET_LINK_LIBRARIES(${NAME} ${PUBLIC_KEYWORD} ${LIB})
ENDMACRO(DEFINE_UNIT_TEST)

MACRO(DYNAMIC_GRAPH_TEST NAME)
  ADD_UNIT_TEST(${NAME} ${NAME}.cpp)
  TARGET_INCLUDE_DIRECTORIES(${NAME} SYSTEM PUBLIC ${EIGEN3_INCLUDE_DIR})
  TARGET_LINK_LIBRARIES(${NAME} ${PROJECT_NAME} ${Boost_LIBRARIES})
ENDMACRO(DYNAMIC_GRAPH_TEST)

SET(CMAKE_VERBOSE_MAKEFILE True)
# Signal cast test.
SET(signalcast_libs signal-cast-registerer-libA signal-cast-registerer-libB)

MESSAGE(status "Eigen3:${EIGEN3_INCLUDE_DIR}")

FOREACH(lib ${signalcast_libs})
  ADD_LIBRARY(${lib} SHARED ${lib}.cpp)
  TARGET_LINK_LIBRARIES(${lib} ${PROJECT_NAME})
ENDFOREACH()

DYNAMIC_GRAPH_TEST(signal-cast-registerer)

# Unit testing.
IF(NOT APPLE)
  DYNAMIC_GRAPH_TEST(entity)
ENDIF(NOT APPLE)
DYNAMIC_GRAPH_TEST(custom-entity)
DYNAMIC_GRAPH_TEST(factory)
DYNAMIC_GRAPH_TEST(pool)
DYNAMIC_GRAPH_TEST(signal-time-dependent)
DYNAMIC_GRAPH_TEST(value)
DYNAMIC_GRAPH_TEST(signal-ptr)
DYNAMIC_GRAPH_TEST(real-time-logger)
DYNAMIC_GRAPH_TEST(debug-trace)
DYNAMIC_GRAPH_TEST(debug-tracer)
TARGET_LINK_LIBRARIES(debug-tracer tracer)
DYNAMIC_GRAPH_TEST(debug-real-time-tracer)
TARGET_LINK_LIBRARIES(debug-real-time-tracer tracer-real-time tracer)
DYNAMIC_GRAPH_TEST(debug-logger)
DYNAMIC_GRAPH_TEST(debug-logger-winit)
DYNAMIC_GRAPH_TEST(signal-all)
DYNAMIC_GRAPH_TEST(command-test)
DYNAMIC_GRAPH_TEST(test-mt)
TARGET_LINK_LIBRARIES(test-mt tracer)
DYNAMIC_GRAPH_TEST(exceptions)
