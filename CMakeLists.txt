cmake_minimum_required(VERSION 2.8.7)
project(FLOWSTAR C CXX)

set(CMAKE_COLOR_MAKEFILE ON)
enable_testing()

# Added for CTest
include(CTest)

set(FLOWSTAR_EXTRA_LINKER_FLAGS "")
set(FLOWSTAR_EXTRA_CXX_FLAGS "")

################################################################
# Set Module Path
################################################################
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/Modules/")

################################################################
# Initialize CXXFLAGS.
################################################################
set(CMAKE_CXX_FLAGS                "-Wall -Wextra -std=c++11 ${FLOWSTAR_EXTRA_CXX_FLAGS}")
set(CMAKE_CXX_FLAGS_DEBUG          "-g -DFLOWSTAR_DEBUG -DFLOWSTAR_TRACE -fno-inline")
set(CMAKE_CXX_FLAGS_MINSIZEREL     "-Os -DNDEBUG")
set(CMAKE_CXX_FLAGS_RELEASE        "-O3 -DNDEBUG")
set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O2 -g")

################################################################
# Compiler-specific C++11 activation.
################################################################
if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU")
  execute_process(
    COMMAND ${CMAKE_CXX_COMPILER} -dumpversion OUTPUT_VARIABLE GCC_VERSION)
  if (NOT (GCC_VERSION VERSION_GREATER 4.8 OR GCC_VERSION VERSION_EQUAL 4.8))
    message(FATAL_ERROR "${PROJECT_NAME} requires g++ 4.8 or greater.")
  endif ()
elseif ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-mismatched-tags -Qunused-arguments")
  if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    # In OSX, clang requires "-stdlib=libc++" to support C++11
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
    set(FLOWSTAR_EXTRA_LINKER_FLAGS "-stdlib=libc++")
  endif ()
else ()
  message(FATAL_ERROR "Your C++ compiler does not support C++11.")
endif ()

################################################################
# Fix Paths
################################################################
SET(CMAKE_SYSTEM_INCLUDE_PATH ${CMAKE_SYSTEM_INCLUDE_PATH}
  /usr/local/include /opt/local/include
  /usr/include/w32api
  /usr/X11R6/include /usr/include/X11
  /usr/pkg/include
  /opt/csw/include /opt/include
  /usr/include /include
  )
SET(CMAKE_SYSTEM_LIBRARY_PATH ${CMAKE_SYSTEM_LIBRARY_PATH}
  $ENV{LD_LIBRARY_PATH}
  /usr/local/lib /opt/local/lib
  /usr/lib/w32api
  /usr/X11R6/lib /usr/lib/X11
  /usr/pkg/lib
  /opt/csw/lib /opt/lib
  /usr/lib /lib
  )
SET(CMAKE_SYSTEM_PROGRAM_PATH ${CMAKE_SYSTEM_PROGRAM_PATH}
  /usr/local/bin /usr/pkg/bin /usr/bin /sbin /bin
  )
SET(CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES
  ${CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES}
  /usr/lib64 /usr/lib32 /usr/lib /lib
  )

################################################################
# Packages
################################################################
# Bison & Flex
find_package(BISON REQUIRED)
find_package(FLEX  REQUIRED)
find_package(MPFR  REQUIRED)
include_directories(${MPFR_INCLUDES})
set(EXTRA_LIBS ${EXTRA_LIBS} ${MPFR_LIBRARIES})
find_package(GMP   REQUIRED)
include_directories(${GMP_INCLUDE_DIR})
set(EXTRA_LIBS ${EXTRA_LIBS} ${GMP_LIBRARIES})
find_package(GSL   REQUIRED)
include_directories(${GSL_INCLUDE_DIR})
set(EXTRA_LIBS ${EXTRA_LIBS} ${GSL_LIBRARIES})
find_package(GLPK  REQUIRED)
include_directories(${GLPK_INCLUDE_DIR})
set(EXTRA_LIBS ${EXTRA_LIBS} ${GLPK_LIBRARIES})

function(JOIN VALUES GLUE OUTPUT)
  string (REGEX REPLACE "([^\\]|^);" "\\1${GLUE}" _TMP_STR "${VALUES}")
  string (REGEX REPLACE "[\\](.)" "\\1" _TMP_STR "${_TMP_STR}") #fixes escaping
  set (${OUTPUT} "${_TMP_STR}" PARENT_SCOPE)
endfunction()

JOIN("${CMAKE_CXX_FLAGS}" "," EXTERNAL_CXX_FLAGS)
JOIN("${FLOWSTAR_EXTRA_LINKER_FLAGS}" "," EXTERNAL_LD_FLAGS)

################################################################
# Include Directories
################################################################
include_directories(${FLOWSTAR_SOURCE_DIR})
include_directories("${FLOWSTAR_BINARY_DIR}")

################################################################
# PARSER
################################################################
# Create custom command for flex/lex (note the outputs)
ADD_CUSTOM_COMMAND(
  COMMAND ${FLEX_EXECUTABLE}
  ARGS -o ${FLOWSTAR_BINARY_DIR}/modelLexer.cc
  ${FLOWSTAR_SOURCE_DIR}/modelLexer.l
  DEPENDS ${FLOWSTAR_SOURCE_DIR}/modelLexer.l
  OUTPUT  ${FLOWSTAR_BINARY_DIR}/modelLexer.cc
  )

# Create custom command for bison/yacc (note the DEPENDS)
ADD_CUSTOM_COMMAND(
  COMMAND ${BISON_EXECUTABLE}
  ARGS -d -y ${FLOWSTAR_SOURCE_DIR}/modelParser.y
  -o ${FLOWSTAR_BINARY_DIR}/modelParser.tab.cpp
  DEPENDS ${FLOWSTAR_BINARY_DIR}/modelLexer.cc
  DEPENDS ${FLOWSTAR_SOURCE_DIR}/modelParser.y
  OUTPUT  ${FLOWSTAR_BINARY_DIR}/modelParser.tab.cpp
  )
# Since parser.c does not exists yet when cmake is run, mark
# it as generated
SET_SOURCE_FILES_PROPERTIES(${FLOWSTAR_BINARY_DIR}/modelParser.tab.cpp GENERATED)
SET_SOURCE_FILES_PROPERTIES(${FLOWSTAR_BINARY_DIR}/modelLexer.cc GENERATED)
################################################################

SET(FLOWSTAR_SRCS
  Interval.cpp
  Matrix.cpp
  Monomial.cpp
  Polynomial.cpp
  TaylorModel.cpp
  Continuous.cpp
  Geometry.cpp
  Constraints.cpp
  Hybrid.cpp)

add_library(flowstar ${FLOWSTAR_SRCS})
add_executable(flowstar-bin modelParser.cpp modelParser.tab.cpp modelLexer.cc)
SET_TARGET_PROPERTIES(flowstar-bin PROPERTIES OUTPUT_NAME flowstar)
target_link_libraries(flowstar-bin flowstar ${EXTRA_LIBS})
