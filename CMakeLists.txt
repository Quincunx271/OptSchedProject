cmake_minimum_required(VERSION 3.4.3)

project(OptSchedPrj)

option(OPTSCHEDPRJ_FLANG "If it is checked-out, build flang." ON)
option(OPTSCHEDPRJ_IGNORE_FLANG "Ignore warnings about flang not existing" OFF)
set(OPTSCHEDPRJ_FLANG_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX} CACHE PATH "The directory to install flang")

set(OPTSCHEDPRJ_FLANG_BUILD_TYPE Release CACHE STRING "The build type for flang")
set(OPTSCHEDPRJ_FLANG_LLVM_BUILD_TYPE ${OPTSCHEDPRJ_FLANG_BUILD_TYPE} CACHE STRING "The build type for flang-llvm")
set(OPTSCHEDPRJ_FLANG_DRIVER_BUILD_TYPE ${OPTSCHEDPRJ_FLANG_BUILD_TYPE} CACHE STRING "The build type for flang-driver")
set(OPTSCHEDPRJ_FLANG_OPENMP_BUILD_TYPE ${OPTSCHEDPRJ_FLANG_BUILD_TYPE} CACHE STRING "The build type for openmp")
set(OPTSCHEDPRJ_FLANG_LIBPGMATH_BUILD_TYPE ${OPTSCHEDPRJ_FLANG_BUILD_TYPE} CACHE STRING "The build type for libpgmath")
set(OPTSCHEDPRJ_FLANG_COMPILER_BUILD_TYPE ${OPTSCHEDPRJ_FLANG_BUILD_TYPE} CACHE STRING "The build type for the flang compiler")

set(OPTSCHEDPRJ_FLANG_CMAKE_GENERATOR ${CMAKE_GENERATOR} CACHE STRING "The generator to use to build flang")
set(OPTSCHEDPRJ_FLANG_LLVM_CMAKE_GENERATOR ${OPTSCHEDPRJ_FLANG_CMAKE_GENERATOR} CACHE STRING "The generator to use to build flang-llvm")
set(OPTSCHEDPRJ_FLANG_DRIVER_CMAKE_GENERATOR ${OPTSCHEDPRJ_FLANG_CMAKE_GENERATOR} CACHE STRING "The generator to use to build flang-driver")
set(OPTSCHEDPRJ_FLANG_OPENMP_CMAKE_GENERATOR ${OPTSCHEDPRJ_FLANG_CMAKE_GENERATOR} CACHE STRING "The generator to use to build openmp")
set(OPTSCHEDPRJ_FLANG_LIBPGMATH_CMAKE_GENERATOR ${OPTSCHEDPRJ_FLANG_CMAKE_GENERATOR} CACHE STRING "The generator to use to build libpgmath")
set(OPTSCHEDPRJ_FLANG_COMPILER_CMAKE_GENERATOR ${OPTSCHEDPRJ_FLANG_CMAKE_GENERATOR} CACHE STRING "The generator to use to build the flang compiler")

include(CMakeParseArguments)
include(ExternalProject)

function(add_variable OUT_LIST NAME)
  set(var "${${OUT_LIST}}")
  
  if(DEFINED ${NAME})
    get_property(type CACHE ${NAME} PROPERTY TYPE)
    if(NOT type)
      set(type UNINITIALIZED)
    endif()
    list(APPEND var -D${NAME}:${type}=${${NAME}})
  endif()

  set(${OUT_LIST} "${var}" PARENT_SCOPE)
endfunction()

function(add_variables OUT_LIST)
  set(list)
  
  foreach(var IN LISTS ARGN)
    add_variable(list ${var})
  endforeach()

  set(${OUT_LIST} "${list}" PARENT_SCOPE)
endfunction()

set(ep_cache_args)
add_variables(ep_cache_args
  CMAKE_CXX_COMPILER_LAUNCHER
  CMAKE_C_COMPILER_LAUNCHER
)

function(SetupProject NAME)
  set(options INSTALL)
  set(args SOURCE_DIR INSTALL_DIR CMAKE_GENERATOR)
  set(listArgs DEPENDS CONFIGURE_ARGS)
  cmake_parse_arguments(ARG "${options}" "${args}" "${listArgs}" ${ARGN})

  if(NOT ARG_SOURCE_DIR)
    set(ARG_SOURCE_DIR "${NAME}")
  endif()

  if(NOT ARG_INSTALL_DIR)
    set(ARG_INSTALL_DIR ${CMAKE_PREFIX_PATH})
  endif()

  if(NOT ARG_CMAKE_GENERATOR)
    set(ARG_CMAKE_GENERATOR ${CMAKE_GENERATOR})
  endif()

  if(ARG_INSTALL)
    set(INSTALL_CMD "")
  else()
    set(INSTALL_CMD "INSTALL_COMMAND \"\"")
  endif()

  ExternalProject_Add(${NAME}
    DEPENDS ${ARG_DEPENDS}
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/${ARG_SOURCE_DIR}
    INSTALL_DIR ${ARG_INSTALL_DIR}
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
      $<$<BOOL:${CMAKE_CXX_COMPILER_LAUNCHER}>:-DCMAKE_CXX_COMPILER=
      ${ARG_CONFIGURE_ARGS}
    CMAKE_CACHE_DEFAULT_ARGS
      ${ep_cache_args}
    CMAKE_GENERATOR ${ARG_CMAKE_GENERATOR}
    USES_TERMINAL_CONFIGURE 1
    USES_TERMINAL_BUILD 1
    USES_TERMINAL_INSTALL 1
    ${INSTALL_CMD}
  )
endfunction()

SetupProject(llvm SOURCE_DIR llvm-project/llvm
  CONFIGURE_ARGS
    -DLLVM_PARALLEL_LINK_JOBS=${LLVM_PARALLEL_LINK_JOBS}
    -DLLVM_ENABLE_PROJECTS='clang'
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    -DLLVM_TARGETS_TO_BUILD=X86
    -DLLVM_BUILD_TOOLS=ON
    -DLLVM_INCLUDE_TESTS=ON
    -DLLVM_OPTIMIZED_TABLEGEN=ON
)

if(OPTSCHEDPRJ_FLANG)
  if(NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/flang/CMakeLists.txt
      OR NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/flang-driver/CMakeLists.txt
      OR NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/flang-llvm/CMakeLists.txt
      OR NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/openmp/CMakeLists.txt)
    if(NOT OPTSCHEDPRJ_IGNORE_FLANG)
      message(WARNING "Flang is improperly checked-out. Either finish checking out flang, flang-driver,"
        " and flang-llvm; set OPTSCHEDPRJ_FLANG=OFF to disable building flang; or set OPTSCHEDPRJ_IGNORE_FLANG=ON"
        " disable this warning while continuing to auto-detect flang."
      )
    endif()
  else()
    SetupProject(flang-llvm
      CONFIGURE_ARGS
        -DCMAKE_BUILD_TYPE=${OPTSCHEDPRJ_FLANG_LLVM_BUILD_TYPE}
        -DLLVM_TARGETS_TO_BUILD=X86
        -DLLVM_BUILD_TOOLS=ON
        -DLLVM_INCLUDE_TESTS=ON
        -DLLVM_OPTIMIZED_TABLEGEN=ON
      CMAKE_GENERATOR ${OPTSCHEDPRJ_FLANG_LLVM_CMAKE_GENERATOR}
      INSTALL
      INSTALL_DIR ${OPTSCHEDPRJ_FLANG_INSTALL_PREFIX}
    )

    SetupProject(flang-driver
      DEPENDS flang-llvm
      CONFIGURE_ARGS
        -DCMAKE_BUILD_TYPE=${OPTSCHEDPRJ_FLANG_DRIVER_BUILD_TYPE}
        -DLLVM_CONFIG=<INSTALL_DIR>/bin/llvm-config
        -DCLANG_ENABLE_STATIC_ANALYZER=ON
      CMAKE_GENERATOR ${OPTSCHEDPRJ_FLANG_DRIVER_CMAKE_GENERATOR}
      INSTALL
      INSTALL_DIR ${OPTSCHEDPRJ_FLANG_INSTALL_PREFIX}
    )

    SetupProject(openmp
      DEPENDS flang-driver
      CONFIGURE_ARGS
        -DCMAKE_BUILD_TYPE=${OPTSCHEDPRJ_FLANG_OPENMP_BUILD_TYPE}
        -DCMAKE_CXX_COMPILER=<INSTALL_DIR>/bin/clang++
        -DCMAKE_C_COMPILER=<INSTALL_DIR>/bin/clang
      CMAKE_GENERATOR ${OPTSCHEDPRJ_FLANG_OPENMP_CMAKE_GENERATOR}
      INSTALL
      INSTALL_DIR ${OPTSCHEDPRJ_FLANG_INSTALL_PREFIX}
    )

    SetupProject(libpgmath SOURCE_DIR flang/runtime/libpgmath
      DEPENDS openmp
      CONFIGURE_ARGS
        -DCMAKE_BUILD_TYPE=${OPTSCHEDPRJ_FLANG_LIBPGMATH_BUILD_TYPE}
        -DCMAKE_CXX_COMPILER=<INSTALL_DIR>/bin/clang++
        -DCMAKE_C_COMPILER=<INSTALL_DIR>/bin/clang
        -DCMAKE_Fortran_COMPILER=<INSTALL_DIR>/bin/flang
      CMAKE_GENERATOR ${OPTSCHEDPRJ_FLANG_LIBPGMATH_CMAKE_GENERATOR}
      INSTALL
      INSTALL_DIR ${OPTSCHEDPRJ_FLANG_INSTALL_PREFIX}
    )

    SetupProject(flang
      DEPENDS libpgmath
      CONFIGURE_ARGS
        -DCMAKE_BUILD_TYPE=${OPTSCHEDPRJ_FLANG_COMPILER_BUILD_TYPE}
        -DCMAKE_CXX_COMPILER=<INSTALL_DIR>/bin/clang++
        -DCMAKE_C_COMPILER=<INSTALL_DIR>/bin/clang
        -DCMAKE_Fortran_COMPILER=<INSTALL_DIR>/bin/flang
        -DLLVM_CONFIG=<INSTALL_DIR>/bin/llvm-config
      CMAKE_GENERATOR ${OPTSCHEDPRJ_FLANG_COMPILER_CMAKE_GENERATOR}
      INSTALL
      INSTALL_DIR ${OPTSCHEDPRJ_FLANG_INSTALL_PREFIX}
    )
  endif()
endif()
