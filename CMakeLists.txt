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

include(CMakeParseArguments)

function(SetupProject NAME)
  set(options INSTALL)
  set(args SOURCE_DIR)
  set(listArgs CONFIGURE_ARGS)
  cmake_parse_arguments(ARG "${options}" "${args}" "${listArgs}" ${ARGN})

  string(REPLACE ";" "\n" cmake_cfg_args "${ARG_CONFIGURE_ARGS}")

  if(NOT ARG_SOURCE_DIR)
    set(ARG_SOURCE_DIR "${NAME}")
  endif()

  if(ARG_INSTALL)
    set(INSTALL_CMD "")
  else()
    set(INSTALL_CMD "INSTALL_COMMAND \"\"")
  endif()

  file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/${NAME}-ep/CMakeLists.txt
    "cmake_minimum_required(VERSION ${CMAKE_VERSION})\n"
    "project(${NAME}_ExternalProject)\n"
    "include(ExternalProject)\n"
    "\n"
    "ExternalProject_Add(${NAME}\n"
    "  SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/${ARG_SOURCE_DIR}\n"
    "  BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/${NAME}-ep/build\n"
    "  INSTALL_DIR ${OPTSCHEDPRJ_FLANG_INSTALL_PREFIX}\n"
    "\n"
    "  CMAKE_ARGS\n"
    "    -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>\n"
    "    \n${cmake_cfg_args}\n"
    "  USES_TERMINAL_CONFIGURE 1\n"
    "  USES_TERMINAL_BUILD 1\n"
    "  USES_TERMINAL_INSTALL 1\n"
    "  ${INSTALL_CMD}\n"
    ")\n"
  )

  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${NAME}-ep/sp-build
    COMMAND
      ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_BINARY_DIR}/${NAME}-ep/sp-build
  )
  add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${NAME}-ep/configure.timestamp
    DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${NAME}-ep/sp-build
    COMMAND
      ${CMAKE_COMMAND}
        -G${CMAKE_GENERATOR}
        -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
        -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
        ${CMAKE_CURRENT_BINARY_DIR}/${NAME}-ep
    COMMAND
      ${CMAKE_COMMAND}
        -E touch ${CMAKE_CURRENT_BINARY_DIR}/${NAME}-ep/configure.timestamp
    COMMENT "Configuring ${NAME}"
    USES_TERMINAL
    WORKING_DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}/${NAME}-ep/sp-build
  )
  add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${NAME}-ep/build.timestamp
    DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${NAME}-ep/configure.timestamp
    COMMAND
      ${CMAKE_COMMAND}
        --build
        ${CMAKE_CURRENT_BINARY_DIR}/${NAME}-ep/sp-build
    COMMAND
      ${CMAKE_COMMAND}
        -E touch ${CMAKE_CURRENT_BINARY_DIR}/${NAME}-ep/build.timestamp
    COMMENT "Building ${NAME}"
    USES_TERMINAL
  )
  add_custom_target(${NAME} ALL
    DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${NAME}-ep/build.timestamp
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
      INSTALL
    )

    SetupProject(flang-driver
      CONFIGURE_ARGS
        -DCMAKE_BUILD_TYPE=${OPTSCHEDPRJ_FLANG_DRIVER_BUILD_TYPE}
        -DLLVM_CONFIG=<INSTALL_DIR>/bin/llvm-config
        -DCLANG_ENABLE_STATIC_ANALYZER=ON
      INSTALL
    )
    add_dependencies(flang-driver flang-llvm)

    SetupProject(openmp 
      CONFIGURE_ARGS
        -DCMAKE_BUILD_TYPE=${OPTSCHEDPRJ_FLANG_OPENMP_BUILD_TYPE}
        -DCMAKE_CXX_COMPILER=<INSTALL_DIR>/bin/clang++
        -DCMAKE_C_COMPILER=<INSTALL_DIR>/bin/clang
      INSTALL
    )
    add_dependencies(openmp flang-driver)

    SetupProject(libpgmath SOURCE_DIR flang/runtime/libpgmath
      CONFIGURE_ARGS
        -DCMAKE_BUILD_TYPE=${OPTSCHEDPRJ_FLANG_LIBPGMATH_BUILD_TYPE}
        -DCMAKE_CXX_COMPILER=<INSTALL_DIR>/bin/clang++
        -DCMAKE_C_COMPILER=<INSTALL_DIR>/bin/clang
        -DCMAKE_Fortran_COMPILER=<INSTALL_DIR>/bin/flang
      INSTALL
    )
    add_dependencies(libpgmath openmp)

    SetupProject(flang 
      CONFIGURE_ARGS
        -DCMAKE_BUILD_TYPE=${OPTSCHEDPRJ_FLANG_COMPILER_BUILD_TYPE}
        -DCMAKE_CXX_COMPILER=<INSTALL_DIR>/bin/clang++
        -DCMAKE_C_COMPILER=<INSTALL_DIR>/bin/clang
        -DCMAKE_Fortran_COMPILER=<INSTALL_DIR>/bin/flang
        -DLLVM_CONFIG=<INSTALL_DIR>/bin/llvm-config
      INSTALL
    )
    add_dependencies(flang libpgmath)
  endif()
endif()
