# =====================================================================
# @file CMakeLists.txt
# @brief for cmake
# @author Pi-Yueh Chuang (pychuang@gwu.edu)
# @version alpha
# @date 2015-09-01
# =====================================================================


CMAKE_MINIMUM_REQUIRED(VERSION 3.0.2)


# =====================================================================
# Cache variables
# =====================================================================

# set default c++ compilers
SET(CMAKE_CXX_COMPILER "mpicxx" CACHE STRING "")

# set default c compilers
SET(CMAKE_C_COMPILER "mpicc" CACHE STRING "")

# set default flags for release and debug mode
SET(CMAKE_CXX_FLAGS_DEBUG "-g" CACHE STRING "")
SET(CMAKE_CXX_FLAGS_RELEASE "-O3 -DNDEBUG" CACHE STRING "")
SET(CMAKE_C_FLAGS_DEBUG "-g" CACHE STRING "")
SET(CMAKE_C_FLAGS_RELEASE "-O3 -DNDEBUG" CACHE STRING "")
SET(CMAKE_SHARED_LINKER_FLAGS "-Wl,--no-undefined" CACHE STRING "")

# set default build type
SET(CMAKE_BUILD_TYPE RELEASE CACHE STRING "")

SET(PETSC_DIR "" CACHE PATH "The path to PETSc.")
SET(PETSC_ARCH "" CACHE STRING "The specific build of PETSc under PETSC_DIR.")
SET(CUDA_DIR "" CACHE PATH "The path to CUDA 6.5.")
SET(AMGX_DIR "" CACHE PATH "The path to AmgX.")


# =====================================================================
# Project settings
# =====================================================================

MESSAGE("")
MESSAGE("====================================")
MESSAGE("Begin configuration")
MESSAGE("====================================")

# start generator
PROJECT(solveFromFiles C CXX)

# use GNU standard installation folder heirarchy
INCLUDE(GNUInstallDirs)

# set the sub folders in build folder
SET(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_BINDIR})

# force the project to use C++ 11
SET(CMAKE_CXX_STANDARD 11)

# other cmake setting
SET(CMAKE_VERBOSE_MAKEFILE ON)
SET(CMAKE_COLOR_MAKEFILE ON)

# add cmake-modules to CMake module searching path
LIST(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/../../cmake-modules")


# =====================================================================
# PETSc
# =====================================================================

MESSAGE("")
MESSAGE("-- Finding PETSc")

FIND_PACKAGE(PETSc REQUIRED)

IF(${PETSc_FOUND})
    MESSAGE("-- Finding PETSc - Success")
ENDIF()


# =====================================================================
# CUDA 6.5
# =====================================================================

MESSAGE("")
MESSAGE("-- Finding CUDA 6.5")

SET(CUDA_TOOLKIT_ROOT_DIR ${CUDA_DIR} CACHE PATH "" FORCE)

FIND_PACKAGE(CUDA 6.5 EXACT REQUIRED)

IF(${CUDA_FOUND})
    SET(CUDA_LIBRARY_DIRS ${CUDA_TOOLKIT_ROOT_DIR}/lib64)
    MESSAGE("-- Finding CUDA 6.5 - Success")
ENDIF()


# =====================================================================
# AmgX
# =====================================================================

MESSAGE("")
MESSAGE("-- Finding AmgX")

MESSAGE("-- Finding amgx_c.h")
FIND_PATH(AMGX_INCLUDE_DIRS 
          NAMES amgx_c.h  
          PATHS ${AMGX_DIR} $ENV{AMGX_DIR} 
          PATH_SUFFIXES "include" "examples")

IF(${AMGX_INCLUDE_DIRS} STREQUAL "AMGX_INCLUDE_DIRS-NOTFOUND")
    MESSAGE(FATAL_ERROR "Could not find amgx_c.h!!!")
ELSE()
    MESSAGE("-- Finding amgx_c.h - found in " ${AMGX_INCLUDE_DIRS})
ENDIF()


MESSAGE("-- Finding amgx_capi.h")
FIND_FILE(AMGX_CAPI_FOUND 
          NAMES amgx_capi.h 
          PATHS ${AMGX_INCLUDE_DIRS} 
          NO_DEFAULT_PATH)

IF(${AMGX_CAPI_FOUND} STREQUAL "AMGX_CAPI_FOUND-NOTFOUND")
    MESSAGE(FATAL_ERROR "amgx_capi.h was not found in: " ${AMGX_INCLUDE_DIRS})
ELSE()
    MESSAGE("-- Finding amgx_capi.h - found")
ENDIF()


MESSAGE("-- Finding amgx_config.h")
FIND_FILE(AMGX_CONFIG_FOUND 
          NAMES amgx_config.h 
          PATHS ${AMGX_INCLUDE_DIRS} 
          NO_DEFAULT_PATH)

IF(${AMGX_CONFIG_FOUND} STREQUAL "AMGX_CONFIG_FOUND-NOTFOUND")
    MESSAGE(FATAL_ERROR "amgx_config.h was not found in: " ${AMGX_INCLUDE_DIRS})
ELSE()
    MESSAGE("-- Finding amgx_config.h - found")
ENDIF()


MESSAGE("-- Finding libamgxsh.so")
FIND_LIBRARY(AMGX_LIBRARIES
             NAMES amgxsh
             PATHS ${AMGX_DIR} $ENV{AMGX_DIR}
             PATH_SUFFIXES "lib")

IF(${AMGX_LIBRARIES} STREQUAL "AMGX_LIBRARIES-NOTFOUND")
    MESSAGE(FATAL_ERROR "libamgxsh.so was not found")
ELSE()
    MESSAGE("-- Finding libamgxsh.so - found: " ${AMGX_LIBRARIES})
ENDIF()

MESSAGE("-- Finding AmgX - Success")

# =====================================================================
# Targets
# =====================================================================

SET(SRC ${CMAKE_SOURCE_DIR}/src)
SET(AmgXWrapperSRC ${CMAKE_SOURCE_DIR}/../../src)

SET(SOURCE
    ${SRC}/main.cpp 
    ${SRC}/createKSP.cpp
    ${SRC}/io.cpp
    ${SRC}/solve.cpp
    ${SRC}/StructArgs.cpp
    ${AmgXWrapperSRC}/init.cpp
    ${AmgXWrapperSRC}/misc.cpp
    ${AmgXWrapperSRC}/setA.cpp
    ${AmgXWrapperSRC}/solve.cpp
    ${AmgXWrapperSRC}/AmgXSolver.cpp)

INCLUDE_DIRECTORIES(
    ${PETSC_INCLUDES} 
    ${AMGX_INCLUDE_DIRS} 
    ${CUDA_INCLUDE_DIRS} 
    ${AmgXWrapperSRC}
    ${SRC})

LINK_DIRECTORIES(
    ${CUDA_LIBRARY_DIRS} 
    ${AMGX_LIBRARY_DIRS}
    ${PETSC_LIBRARY_DIRS})

ADD_EXECUTABLE(solveFromFiles ${SOURCE})

TARGET_LINK_LIBRARIES(solveFromFiles 
    ${PETSC_LIBRARIES} 
    ${AMGX_LIBRARIES} 
    ${CUDA_LIBRARIES})

ADD_CUSTOM_COMMAND(
    TARGET solveFromFiles
    POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E make_directory ${PROJECT_BINARY_DIR}/configs
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/configs ${PROJECT_BINARY_DIR}/configs/)

# =====================================================================
# Print information
# =====================================================================

MESSAGE("")
MESSAGE("====================================")
MESSAGE("Config Information:")
MESSAGE("====================================")
MESSAGE("")

MESSAGE("Build type: " ${CMAKE_BUILD_TYPE})
MESSAGE("Build shared library: " ${BUILD_SHARED_LIBS})
MESSAGE("Doxygen available: " ${DOXYGEN_FOUND})
MESSAGE("Installation path: " ${CMAKE_INSTALL_PREFIX})
MESSAGE("")

MESSAGE("Compilers:")
MESSAGE("\tCXX compiler: " ${CMAKE_CXX_COMPILER})
MESSAGE("\tC compiler: " ${CMAKE_C_COMPILER})
MESSAGE("")

MESSAGE("Flags:")
MESSAGE("\tCXX: " ${CMAKE_CXX_FLAGS_${CMAKE_BUILD_TYPE}})
MESSAGE("")

MESSAGE("PETSc:")
MESSAGE("\tPETSc dir: " ${PETSC_DIR})
MESSAGE("\tHeader path: " ${PETSC_INCLUDES})
MESSAGE("\tLibraries: " ${PETSC_LIBRARIES})
MESSAGE("")

MESSAGE("CUDA:")
MESSAGE("\tCUDA dir: " ${CUDA_TOOLKIT_ROOT_DIR})
MESSAGE("\tVersion: " ${CUDA_VERSION_STRING})
MESSAGE("\tHeader path: " ${CUDA_INCLUDE_DIRS})
MESSAGE("\tLibrary path: " ${CUDA_LIBRARY_DIRS})
MESSAGE("\tLibraries: " ${CUDA_LIBRARIES})
MESSAGE("")

MESSAGE("AmgX:")
MESSAGE("\tAmgX dir: " ${AMGX_DIR})
MESSAGE("\tHeader path: " ${AMGX_INCLUDE_DIRS})
MESSAGE("\tLibraries: " ${AMGX_LIBRARIES})
MESSAGE("")
