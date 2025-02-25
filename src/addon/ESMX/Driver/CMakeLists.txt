cmake_minimum_required(VERSION 3.22)

# load packages and modules
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/cmake)
find_package(Python 3.5 REQUIRED COMPONENTS Interpreter)
find_package(ESMF 8.6.0 MODULE REQUIRED)
include(FetchContent)

# esmx functions
function(esmx_check_ret RETVAL MESSAGE)
  if(NOT RETVAL EQUAL "0")
    message(FATAL_ERROR ${MESSAGE})
  endif()
endfunction()

# unset user ESMF variables stored in cache
get_cmake_property(_variableNames VARIABLES)
foreach (_variableName ${_variableNames})
  if("${_variableName}" MATCHES "^ESMX_*")
    set(${_variableName} ${${_variableName}})
    unset(${_variableName} CACHE)
  endif()
endforeach()

# define ESMX_BUILD_FILE
if(NOT DEFINED ESMX_BUILD_FILE)
  set(ESMX_BUILD_FILE "esmxBuild.yaml")
endif()

# generate app config file(s)
execute_process(COMMAND ${Python_EXECUTABLE}
  ${CMAKE_CURRENT_LIST_DIR}/esmx_app_config.py --ifile ${ESMX_BUILD_FILE} --odir ${CMAKE_CURRENT_BINARY_DIR}
  RESULT_VARIABLE ret)
esmx_check_ret(${ret} "esmx_app_config.py failed processing ${ESMX_BUILD_FILE}")
include(${CMAKE_CURRENT_BINARY_DIR}/appConf.txt)

# disabled components
if(DEFINED ESMX_DISABLE_COMPS)
  string(REPLACE ";" " " ESMX_DISABLE_COMPS "${ESMX_DISABLE_COMPS}")
  string(REPLACE " " "," ESMX_DISABLE_COMPS "${ESMX_DISABLE_COMPS}")
  set(ESMX_DISABLE_COMPS --disable_comps ${ESMX_DISABLE_COMPS})
endif()

# define ESMX_EXE_NAME
if(NOT DEFINED ESMX_EXE_NAME)
  set(ESMX_EXE_NAME "esmx_app")
  set(ESMX_EXE_NAME "esmx_app" PARENT_SCOPE)
else()
  set(ESMX_EXE_NAME ${ESMX_EXE_NAME} PARENT_SCOPE)
endif()

# test options
if(ESMX_TEST)
  enable_testing()
  if(NOT DEFINED ESMX_TEST_DIR)
    set(ESMX_TEST_DIR "${CMAKE_BINARY_DIR}-tests")
  endif()
  file(MAKE_DIRECTORY ${ESMX_TEST_DIR})
  if(NOT DEFINED ESMX_TEST_EXE)
    string(REPLACE "\"" "" ESMX_TEST_EXE ${ESMF_INTERNAL_MPIRUN})
    string(STRIP ${ESMX_TEST_EXE} ESMX_TEST_EXE)
    string(REGEX REPLACE " .*" "" ESMX_TEST_EXE ${ESMX_TEST_EXE})
  endif()
  if(DEFINED ESMX_TEST_TASKS)
    set(ESMX_TEST_TASKS -np ${ESMX_TEST_TASKS})
  else()
    set(ESMX_TEST_TASKS -np 4)
  endif()
endif()

# build verbosity
if(ESMX_CMAKE_BUILD_VERBOSE)
  set(ESMX_CMAKE_BUILD_VERBOSE "-v")
elseif(ESMX_BUILD_VERBOSE)
  set(ESMX_CMAKE_BUILD_VERBOSE "-v")
endif()

# build jobs
if(DEFINED ESMX_CMAKE_BUILD_JOBS)
  set(ESMX_CMAKE_BUILD_JOBS "-j ${ESMX_CMAKE_BUILD_JOBS}")
elseif(DEFINED ESMX_BUILD_JOBS)
  set(ESMX_CMAKE_BUILD_JOBS "-j ${ESMX_BUILD_JOBS}")
endif()
if(DEFINED ESMX_MAKE_BUILD_JOBS)
  set(ESMX_MAKE_BUILD_JOBS "--jobs=${ESMX_MAKE_BUILD_JOBS}")
elseif(DEFINED ESMX_BUILD_JOBS)
  set(ESMX_MAKE_BUILD_JOBS "--jobs=${ESMX_BUILD_JOBS}")
endif()

# compiler options
if(CMAKE_Fortran_COMPILER_ID MATCHES "GNU")
  set(CMAKE_Fortran_FLAGS_RELEASE "-O2")
  set(CMAKE_Fortran_FLAGS_RELWITHDEBINFO "-g -fbacktrace -O2")
  set(CMAKE_Fortran_FLAGS_DEBUG "-g -fbacktrace -O0 -fcheck=all -ffpe-trap=invalid,zero,overflow,underflow")
elseif(CMAKE_Fortran_COMPILER_ID MATCHES "Intel")
  set(CMAKE_Fortran_FLAGS_RELEASE "-O2")
  set(CMAKE_Fortran_FLAGS_RELWITHDEBINFO "-g -traceback -O2")
  set(CMAKE_Fortran_FLAGS_DEBUG "-g -traceback -O0 -check all -fpe0 -ftrapuv -init=snan,arrays")
elseif(CMAKE_Fortran_COMPILER_ID MATCHES "NVHPC")
  set(CMAKE_Fortran_FLAGS_RELEASE "-O2 -fast")
  set(CMAKE_Fortran_FLAGS_RELWITHDEBINFO "-g -traceback -O2 -fast")
  set(CMAKE_Fortran_FLAGS_DEBUG "-g -traceback -O0")
else()
  message(WARNING "${CMAKE_Fortran_COMPILER_ID} Fortran compiler will be used with default options")
endif()

# generate component config file(s)
execute_process(COMMAND ${Python_EXECUTABLE}
  ${CMAKE_CURRENT_LIST_DIR}/esmx_comp_config.py --ifile ${ESMX_BUILD_FILE} --odir ${CMAKE_CURRENT_BINARY_DIR} ${ESMX_DISABLE_COMPS}
  RESULT_VARIABLE ret)
esmx_check_ret(${ret} "esmx_comp_config.py failed processing ${ESMX_BUILD_FILE}")

# generate test config file(s)
execute_process(COMMAND ${Python_EXECUTABLE}
  ${CMAKE_CURRENT_LIST_DIR}/esmx_test_config.py --ifile ${ESMX_BUILD_FILE} --odir ${CMAKE_CURRENT_BINARY_DIR}
  RESULT_VARIABLE ret)
esmx_check_ret(${ret} "esmx_test_config.py failed processing ${ESMX_BUILD_FILE}")

# specify driver dependency
add_library(esmx_driver ESMX_Driver.F90)
target_include_directories(esmx_driver PUBLIC ${CMAKE_CURRENT_BINARY_DIR})
target_link_libraries(esmx_driver PUBLIC ESMF)

# add module paths
foreach(ESMX_CMAKE_MODULE_PATH IN ITEMS ${ESMX_CMAKE_MODULE_PATHS})
  list(APPEND CMAKE_MODULE_PATH ${ESMX_CMAKE_MODULE_PATH})
endforeach()

# link packages
foreach(ESMX_LINK_PACKAGE IN ITEMS ${ESMX_LINK_PACKAGES})
  find_package(${ESMX_LINK_PACKAGE} REQUIRED)
endforeach()

# link external libraries
foreach(ESMX_LINK_LIBRARY IN ITEMS ${ESMX_LINK_LIBRARIES})
  if(TARGET ${ESMX_LINK_LIBRARY})
    target_link_libraries(esmx_driver PUBLIC ${ESMX_LINK_LIBRARY})
  else()
    find_library(FND_LINK_LIBRARY
      NAMES ${ESMX_LINK_LIBRARY}
      HINTS ${ESMX_LINK_PATHS}
      PATH_SUFFIXES "." "lib" "lib64"
      NO_CACHE
    )
    if(NOT FND_LINK_LIBRARY)
      message(FATAL_ERROR "Cannot find link_libraries ${ESMX_LINK_LIBRARY} in ${ESMX_LINK_PATHS}")
    endif()
    target_link_libraries(esmx_driver PUBLIC ${FND_LINK_LIBRARY})
    unset(FND_LINK_LIBRARY)
  endif()
endforeach()

# add components
set(CMP_OPTIONS BUILD_TYPE;
                SOURCE_DIR;
                GIT_REPOSITORY;
                GIT_TAG;
                GIT_DIR;
                CMAKE_CONFIG;
                INSTALL_PREFIX;
                LIBRARY_DIR;
                CONFIG_DIR;
                INCLUDE_DIR;
                FORT_MODULE;
                LIBRARIES;
                LINK_LIBRARIES;
                LINK_PATHS;
                BUILD_ARGS;
                BUILD_SCRIPT;
                TEST_DIR;
                TEST_EXE;
                TEST_TASKS)
include(${CMAKE_CURRENT_BINARY_DIR}/compList.txt)
foreach(CMP IN ITEMS ${COMPS})

  # get component options
  foreach(CMPVAR IN LISTS CMP_OPTIONS)
    set(CMP_${CMPVAR} "${${CMP}-${CMPVAR}}")
  endforeach()
  string(TOLOWER "${CMP_BUILD_TYPE}" CMP_BUILD_TYPE)
  if(CMP_GIT_DIR STREQUAL "")
    set(CMP_GIT_DIR ${CMP_SOURCE_DIR})
  endif()
  if(CMP_INSTALL_PREFIX STREQUAL "")
    set(CMP_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX})
  endif()
  string(REPLACE "," ";" "${${CMP}-LIBRARIES}" CMP_LIBRARIES)
  string(REPLACE "," ";" "${${CMP}-LINK_LIBRARIES}" CMP_LINK_LIBRARIES)
  string(REPLACE "," ";" "${${CMP}-LINK_PATHS}" CMP_LINK_PATHS)
  if(CMP_BUILD_SCRIPT STREQUAL "")
    set(CMP_BUILD_SCRIPT compile.sh;compile;build.sh;build)
  endif()

  # download or update git repository
  if(NOT CMP_GIT_REPOSITORY STREQUAL "")
    message(STATUS "Fetching content ... ${CMP}")
    if(CMP_GIT_TAG STREQUAL "")
      message(FATAL_ERROR "Git tag required for ${CMP}")
    endif()
    if(NOT EXISTS ${CMP_SOURCE_DIR})
      set(CMP_GIT_TAG -b ${CMP_GIT_TAG})
      execute_process(
        COMMAND git clone ${CMP_GIT_TAG}
          ${CMP_GIT_REPOSITORY} ${CMP_GIT_DIR}
        RESULT_VARIABLE ret
      )
      esmx_check_ret(${ret} "git clone failed for ${CMP}")
    else()
      execute_process(
        COMMAND git remote set-url origin ${CMP_GIT_REPOSITORY}
        WORKING_DIRECTORY ${CMP_GIT_DIR}
        RESULT_VARIABLE ret
      )
      esmx_check_ret(${ret} "git set-url failed for ${CMP}")
      execute_process(
        COMMAND git fetch origin
        WORKING_DIRECTORY ${CMP_GIT_DIR}
        RESULT_VARIABLE ret
      )
      esmx_check_ret(${ret} "git fetch failed for ${CMP}")
      execute_process(
        COMMAND git checkout ${CMP_GIT_TAG}
        WORKING_DIRECTORY ${CMP_GIT_DIR}
        RESULT_VARIABLE ret
      )
      esmx_check_ret(${ret} "git checkout failed for ${CMP}")
      execute_process(
        COMMAND git pull origin ${CMP_GIT_TAG}
        WORKING_DIRECTORY ${CMP_GIT_DIR}
        RESULT_VARIABLE ret
      )
      esmx_check_ret(${ret} "git pull failed for ${CMP}")
    endif()
  endif()

  # automatically determine build type based on files
  if(${CMP_BUILD_TYPE} STREQUAL "auto")
    # find build configuration or libraries
    find_file(FND_CMAKE_LISTS
      NAMES CMakeLists.txt cmakelists.txt CMAKELISTS.txt
      PATHS ${CMP_SOURCE_DIR}
      NO_CACHE NO_DEFAULT_PATH
    )
    find_file(FND_MAKE_FILE
      NAMES Makefile makefile MAKEFILE
      PATHS ${CMP_SOURCE_DIR}
      NO_CACHE NO_DEFAULT_PATH
    )
    find_file(FND_BUILD_SCRIPT
      NAMES ${CMP_BUILD_SCRIPT}
      PATHS ${CMP_SOURCE_DIR}
      NO_CACHE NO_DEFAULT_PATH
    )
    find_file(FND_CMAKE_CONFIG
      NAMES ${CMP_CMAKE_CONFIG}
      HINTS ${CMP_INSTALL_PREFIX}
      PATH_SUFFIXES ${CMP_CONFIG_DIR} "." "cmake" "config"
      NO_CACHE NO_DEFAULT_PATH
    )
    set(FND_LIBRARIES TRUE)
    foreach(CMP_LIBRARY IN ITEMS ${CMP_LIBRARIES})
      if(NOT TARGET ${CMP_LIBRARY})
        find_library(FND_LIBRARY
          NAMES ${CMP_LIBRARY}
          HINTS ${CMP_INSTALL_PREFIX}
          PATH_SUFFIXES ${CMP_LIBRARY_DIR} "." "lib" "lib64"
          NO_CACHE NO_DEFAULT_PATH
        )
        if(NOT FND_LIBRARY)
          set(FND_LIBRARIES FALSE)
        endif()
        unset(FND_LIBRARY)
      endif()
    endforeach()
    if(FND_CMAKE_LISTS)
      set(CMP_BUILD_TYPE "cmake")
    elseif(FND_MAKE_FILE)
      set(CMP_BUILD_TYPE "make")
    elseif(FND_BUILD_SCRIPT)
      set(CMP_BUILD_TYPE "script")
    elseif(FND_CMAKE_CONFIG OR FND_LIBRARIES)
      set(CMP_BUILD_TYPE "none")
    else()
      message(FATAL_ERROR "Cannot find build configuration for ${CMP}"
        " in ${CMP_SOURCE_DIR}")
    endif()
    unset(FND_CMAKE_LISTS)
    unset(FND_MAKE_FILE)
    unset(FND_BUILD_SCRIPT)
    unset(FND_CMAKE_CONFIG)
    unset(FND_LIBRARIES)
  endif()

  # build component with cmake
  if(${CMP_BUILD_TYPE} STREQUAL "cmake")
    message(STATUS "ESMX building (CMAKE) - ${CMP} - ${CMP_SOURCE_DIR}")
    find_file(FND_CMAKE_LISTS
      NAMES CMakeLists.txt cmakelists.txt CMAKELISTS.txt
      PATHS ${CMP_SOURCE_DIR}
      NO_CACHE NO_DEFAULT_PATH
    )
    if(NOT FND_CMAKE_LISTS)
      message(FATAL_ERROR "Could not find CMakeLists.txt"
        " in ${CMP_SOURCE_DIR}."
        " Check source_dir in build configuration."
      )
    endif()
    unset(FND_CMAKE_LISTS)
    file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/${CMP})
    execute_process(
      COMMAND ${CMAKE_COMMAND} -S${CMP_SOURCE_DIR}
        ${ESMX_CMAKE_BUILD_ARGS}
        ${ESMX_BUILD_ARGS}
        ${CMP_BUILD_ARGS}
        -DCMAKE_INSTALL_PREFIX=${CMP_INSTALL_PREFIX}
        -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
      WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/${CMP}
      RESULT_VARIABLE ret
    )
    esmx_check_ret(${ret} "cmake generate failed for ${CMP}")
    execute_process(
      COMMAND ${CMAKE_COMMAND} --build .
        ${ESMX_CMAKE_BUILD_VERBOSE}
        ${ESMX_CMAKE_BUILD_JOBS}
      WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/${CMP}
      RESULT_VARIABLE ret
    )
    esmx_check_ret(${ret} "cmake build failed for ${CMP}")
    execute_process(
      COMMAND ${CMAKE_COMMAND} --install .
      WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/${CMP}
      RESULT_VARIABLE ret
    )
    esmx_check_ret(${ret} "cmake install failed for ${CMP}")

  # build component with make
  elseif(${CMP_BUILD_TYPE} STREQUAL "make")
    message(STATUS "ESMX building (MAKE) - ${CMP} - ${CMP_SOURCE_DIR}")
    find_file(FND_MAKE_FILE
      NAMES Makefile makefile MAKEFILE
      PATHS ${CMP_SOURCE_DIR}
      NO_CACHE NO_DEFAULT_PATH
    )
    if(NOT FND_MAKE_FILE)
      message(FATAL_ERROR "Could not find Makefile"
        " in ${CMP_SOURCE_DIR}."
        " Check source_dir in build configuration."
      )
    endif()
    unset(FND_MAKE_FILE)
    execute_process(
      COMMAND ${CMAKE_MAKE_PROGRAM}
        ${ESMX_MAKE_BUILD_JOBS}
        ${ESMX_MAKE_BUILD_ARGS}
        ${ESMX_BUILD_ARGS}
        ${CMP_BUILD_ARGS}
      WORKING_DIRECTORY ${CMP_SOURCE_DIR}
      RESULT_VARIABLE ret
    )
    esmx_check_ret(${ret} "make build failed for ${CMP}")

  # build component with script
  elseif(${CMP_BUILD_TYPE} STREQUAL "script")
    message(STATUS "ESMX building (SCRIPT) - ${CMP} - ${CMP_SOURCE_DIR}")
    find_file(FND_BUILD_SCRIPT
      NAMES ${CMP_BUILD_SCRIPT}
      PATHS ${CMP_SOURCE_DIR}
      NO_CACHE NO_DEFAULT_PATH
    )
    if(NOT FND_BUILD_SCRIPT)
      message(FATAL_ERROR "Could not find ${CMP_BUILD_SCRIPT}"
        " in ${CMP_SOURCE_DIR}."
        " Check source_dir in build configuration."
      )
    endif()
    execute_process(
      COMMAND ${FND_BUILD_SCRIPT}
        ${ESMX_SCRIPT_BUILD_ARGS}
        ${ESMX_BUILD_ARGS}
        ${CMP_BUILD_ARGS}
      WORKING_DIRECTORY ${CMP_SOURCE_DIR}
      RESULT_VARIABLE ret
    )
    esmx_check_ret(${ret} "script build failed for ${CMP}")
    unset(FND_BUILD_SCRIPT)
  # skip build step
  elseif(${CMP_BUILD_TYPE} STREQUAL "none")
    # No Op
  else()
    message(FATAL_ERROR "Invalid BUILD_TYPE: ${CMP_BUILD_TYPE} for ${CMP}")
  endif()

  # include modules and link libraries
  find_file(FND_CMAKE_CONFIG
    NAMES ${CMP_CMAKE_CONFIG}
    HINTS ${CMP_INSTALL_PREFIX} ${CMAKE_BINARY_DIR}/${CMP}
    PATH_SUFFIXES ${CMP_CONFIG_DIR} "." "cmake" "config"
    NO_CACHE NO_DEFAULT_PATH
  )
  if(FND_CMAKE_CONFIG)
    include(${FND_CMAKE_CONFIG})
  endif()
  find_path(FND_FORT_MODULE
    NAMES ${CMP_FORT_MODULE}
    HINTS ${CMP_INSTALL_PREFIX} ${CMAKE_BINARY_DIR}/${CMP}
    PATH_SUFFIXES ${CMP_INCLUDE_DIR} "." "include" "mod"
    NO_CACHE NO_DEFAULT_PATH
  )
  if(FND_FORT_MODULE)
    target_include_directories(esmx_driver PUBLIC ${FND_FORT_MODULE})
  elseif(NOT FND_CMAKE_CONFIG)
    message(FATAL_ERROR "Cannot find fort_module ${CMP_FORT_MODULE} in ${CMP_INSTALL_PREFIX}")
  endif()
  unset(FND_FORT_MODULE)
  unset(FND_CMAKE_CONFIG)
  foreach(CMP_LIBRARY IN ITEMS ${CMP_LIBRARIES})
    if(TARGET ${CMP_LIBRARY})
      target_link_libraries(esmx_driver PUBLIC ${CMP_LIBRARY})
    else()
      find_library(FND_LIBRARY
        NAMES ${CMP_LIBRARY}
        HINTS ${CMP_INSTALL_PREFIX} ${CMAKE_BINARY_DIR}/${CMP}
        PATH_SUFFIXES ${CMP_LIBRARY_DIR} "." "lib" "lib64"
        NO_CACHE NO_DEFAULT_PATH
      )
      if(NOT FND_LIBRARY)
        message(FATAL_ERROR "Cannot find libraries ${CMP_LIBRARY} in ${CMP_INSTALL_PREFIX}")
      endif()
      target_link_libraries(esmx_driver PUBLIC ${FND_LIBRARY})
      unset(FND_LIBRARY)
    endif()
  endforeach()

  # link external libraries
  foreach(CMP_LINK_LIBRARY IN ITEMS ${CMP_LINK_LIBRARIES})
    if(TARGET ${CMP_LINK_LIBRARY})
      target_link_libraries(esmx_driver PUBLIC ${CMP_LINK_LIBRARY})
    else()
      find_library(FND_LINK_LIBRARY
        NAMES ${CMP_LINK_LIBRARY}
        HINTS ${CMP_LINK_PATHS}
        PATH_SUFFIXES "." "lib" "lib64"
        NO_CACHE
      )
      if(NOT FND_LINK_LIBRARY)
        message(FATAL_ERROR "Cannot find link_libraries ${CMP_LINK_LIBRARY} in ${CMP_LINK_PATHS}")
      endif()
      target_link_libraries(esmx_driver PUBLIC ${FND_LINK_LIBRARY})
      unset(FND_LINK_LIBRARY)
    endif()
  endforeach()

  # add component test
  if(ESMX_TEST)
    if(CMP_TEST_DIR STREQUAL "")
      message(WARNING "No test directory provided for ${CMP}")
    elseif(NOT EXISTS ${CMP_TEST_DIR})
      message(FATAL_ERROR "Cannot find test directory ${CMP_TEST_DIR}")
    else()
      if(CMP_TEST_EXE STREQUAL "")
        set(CMP_TEST_EXE ${ESMX_TEST_EXE})
      endif()
      if(CMP_TEST_TASKS STREQUAL "")
        set(CMP_TEST_TASKS ${ESMX_TEST_TASKS})
      else()
        set(CMP_TEST_TASKS -np ${CMP_TEST_TASKS})
      endif()

      # copy test files
      file(COPY ${CMP_TEST_DIR}/ DESTINATION ${ESMX_TEST_DIR}/cmp/${CMP})
      file(CREATE_LINK ${CMAKE_BINARY_DIR}/${ESMX_EXE_NAME}
        ${ESMX_TEST_DIR}/cmp/${CMP}/${ESMX_EXE_NAME}
        SYMBOLIC
      )

      # find test application
      find_program(FND_TEST_EXE
        NAMES ${CMP_TEST_EXE}
        NO_CACHE
      )
      if(NOT FND_TEST_EXE)
        message(FATAL_ERROR "Cannot find test executable: ${CMP_TEST_EXE}")
      endif()

      # add test
      add_test(NAME esmx-cmp-${CMP}
        COMMAND ${FND_TEST_EXE} ${CMP_TEST_TASKS} ./${ESMX_EXE_NAME}
        WORKING_DIRECTORY ${ESMX_TEST_DIR}/cmp/${CMP})
      unset(FND_TEST_EXE)
    endif()
  endif()

  # unset variables used for component
  foreach(CMPVAR IN LISTS CMP_OPTIONS)
    unset(CMP_${CMPVAR})
  endforeach()

endforeach()

# add system tests
if(ESMX_TEST)
  set(TST_OPTIONS  DIR;
                   EXE;
                   TASKS)
  include(${CMAKE_CURRENT_BINARY_DIR}/testList.txt)
  foreach(TST IN ITEMS ${TESTS})

    # get test options
    foreach(TSTVAR IN LISTS TST_OPTIONS)
      set(TST_${TSTVAR} "${${TST}-${TSTVAR}}")
    endforeach()
    if(TST_DIR STREQUAL "")
      message(WARNING "No test directory provided for: ${TST}")
    elseif(NOT EXISTS ${TST_DIR})
      message(FATAL_ERROR "Cannot find test directory: ${TST_DIR}")
    endif()
    if(TST_EXE STREQUAL "")
      set(TST_EXE ${ESMX_TEST_EXE})
    endif()
    if(TST_TASKS STREQUAL "")
      set(TST_TASKS ${ESMX_TEST_TASKS})
    else()
      set(TST_TASKS -np ${TST_TASKS})
    endif()

    # copy test files
    file(COPY ${TST_DIR}/ DESTINATION ${ESMX_TEST_DIR}/sys/${TST})
    file(CREATE_LINK ${CMAKE_BINARY_DIR}/${ESMX_EXE_NAME}
      ${ESMX_TEST_DIR}/sys/${TST}/${ESMX_EXE_NAME}
      SYMBOLIC
    )

    # find test application
    find_program(FND_TST_EXE
      NAMES ${TST_EXE}
      NO_CACHE
    )
    if(NOT FND_TST_EXE)
      message(FATAL_ERROR "Cannot find test executable: ${TST_EXE}")
    endif()

    # add test
    add_test(NAME esmx-sys-${TST}
      COMMAND ${FND_TST_EXE} ${TST_TASKS} ./${ESMX_EXE_NAME}
      WORKING_DIRECTORY ${ESMX_TEST_DIR}/sys/${TST})
    unset(FND_TST_EXE)

    # unset variables used for test
    foreach(TSTVAR IN LISTS TEST_OPTIONS)
      unset(TST_${TSTVAR})
    endforeach()

  endforeach()
endif()

