# CMakeLists.txt

# Copyright (C) 2007-2014 Glenn Randers-Pehrson

# This code is released under the libpng license.
# For conditions of distribution and use, see the disclaimer
# and license in png.h

IF(BIICODE)
  # Initializes block variables
  INIT_BIICODE_BLOCK()
  set(PNG_SHARED OFF)
ENDIF()


cmake_minimum_required(VERSION 2.4.4)
set(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS true)
set(CMAKE_CONFIGURATION_TYPES "Release;Debug;MinSizeRel;RelWithDebInfo")
project(libpng C)
enable_testing()

set(PNGLIB_MAJOR 1)
set(PNGLIB_MINOR 6)
set(PNGLIB_RELEASE 15)
set(PNGLIB_NAME libpng${PNGLIB_MAJOR}${PNGLIB_MINOR})
set(PNGLIB_VERSION ${PNGLIB_MAJOR}.${PNGLIB_MINOR}.${PNGLIB_RELEASE})

if(NOT WIN32)
  find_library(M_LIBRARY
    NAMES m
    PATHS /usr/lib /usr/local/lib
  )
  if(NOT M_LIBRARY)
    message(STATUS
      "math library 'libm' not found - floating point support disabled")
  endif()
else()
  # not needed on windows
  set(M_LIBRARY "")
endif()

# COMMAND LINE OPTIONS
if(DEFINED PNG_SHARED)
  option(PNG_SHARED "Build shared lib" ${PNG_SHARED})
else()
  option(PNG_SHARED "Build shared lib" ON)
endif()

if(DEFINED PNG_STATIC)
  option(PNG_STATIC "Build static lib" ${PNG_STATIC})
else()
  option(PNG_STATIC "Build static lib" ON)
endif()

option(PNG_TESTS  "Build libpng tests" ON)

# Many more configuration options could be added here
option(PNG_DEBUG         "Build with debug output" NO)
option(PNGARG            "Disable ANSI-C prototypes" NO)

# SET LIBNAME
set(PNG_LIB_NAME png${PNGLIB_MAJOR}${PNGLIB_MINOR})

# to distinguish between debug and release lib
set(CMAKE_DEBUG_POSTFIX "d")

# Use the prebuilt pnglibconf.h file from the scripts folder
# TODO: fix this by building with awk; without this no cmake build can be
# configured directly (to do so indirectly use your local awk to build a
# pnglibconf.h in the build directory.)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scripts/pnglibconf.h.prebuilt
               ${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.h)

include_directories(${CMAKE_CURRENT_BINARY_DIR})

# OUR SOURCES
set(libpng_public_hdrs
  png.h
  pngconf.h
  ${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.h
)
set(libpng_sources
  ${libpng_public_hdrs}
  pngdebug.h
  pnginfo.h
  pngpriv.h
  pngstruct.h
  png.c
  pngerror.c
  pngget.c
  pngmem.c
  pngpread.c
  pngread.c
  pngrio.c
  pngrtran.c
  pngrutil.c
  pngset.c
  pngtrans.c
  pngwio.c
  pngwrite.c
  pngwtran.c
  pngwutil.c
)
set(pngtest_sources
  pngtest.c
)
set(pngvalid_sources
  contrib/libtests/pngvalid.c
)
set(pngstest_sources
  contrib/libtests/pngstest.c
)
# SOME NEEDED DEFINITIONS

if(MSVC)
  add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
endif(MSVC)

if(PNG_DEBUG)
  add_definitions(-DPNG_DEBUG)
endif()

# NOW BUILD OUR TARGET
unset(PNG_LIB_TARGETS)

##############################################################
IF(BIICODE)
  BII_FILTER_LIB_SRC(libpng_sources) 

  IF(NOT PNG_TESTS) # Remove exe targets for test if needed
    MESSAGE("EXCLUDING TESTS...")
    FOREACH(loop_var ${BII_BLOCK_EXES})
        if(loop_var MATCHES "contrib(.*)")
          LIST(REMOVE_ITEM BII_BLOCK_EXES ${loop_var})
        endif()
    ENDFOREACH()  
    MESSAGE(${BII_BLOCK_EXES})
  ENDIF()
  
  MESSAGE(${BII_BLOCK_EXES})
  # Excluded examples (TODO: FIX COMPILATION WITH BIICODE)
  LIST(REMOVE_ITEM BII_BLOCK_EXES contrib_gregbook_rpng-x)
  LIST(REMOVE_ITEM BII_BLOCK_EXES contrib_gregbook_rpng2-x)
  LIST(REMOVE_ITEM BII_BLOCK_EXES contrib_gregbook_wpng)
  LIST(REMOVE_ITEM BII_BLOCK_EXES contrib_libtests_tarith)
  
  

  ADD_BIICODE_TARGETS()
  # ADD BINARY DIR TO FIND THE CONFIGURED 
  TARGET_INCLUDE_DIRECTORIES(${BII_BLOCK_TARGET} INTERFACE ${CMAKE_CURRENT_BINARY_DIR})

  ##biicode translation
  IF(M_LIBRARY)
      target_link_libraries(${BII_LIB_TARGET} ${M_LIBRARY})
  ENDIF()
  SET(PNG_LIB_NAME ${BII_LIB_TARGET})
ENDIF()

##############################################################


if(PNG_SHARED)
  add_library(${PNG_LIB_NAME} SHARED ${libpng_sources})
  set(PNG_LIB_TARGETS ${PNG_LIB_NAME})
  if(MSVC)
    # msvc does not append 'lib' - do it here to have consistent name
    set_target_properties(${PNG_LIB_NAME} PROPERTIES PREFIX "lib")
    set_target_properties(${PNG_LIB_NAME} PROPERTIES IMPORT_PREFIX "lib")
  endif()
  target_link_libraries(${PNG_LIB_NAME} ${ZLIB_LIBRARY} ${M_LIBRARY})
endif()

if(PNG_STATIC)
# does not work without changing name
  set(PNG_LIB_NAME_STATIC ${PNG_LIB_NAME}_static)
  add_library(${PNG_LIB_NAME_STATIC} STATIC ${libpng_sources})
  list(APPEND PNG_LIB_TARGETS ${PNG_LIB_NAME_STATIC})
  if(MSVC)
    # msvc does not append 'lib' - do it here to have consistent name
    set_target_properties(${PNG_LIB_NAME_STATIC} PROPERTIES PREFIX "lib")
  endif()
  target_link_libraries(${PNG_LIB_NAME_STATIC} ${ZLIB_LIBRARY} ${M_LIBRARY})
endif()

if(NOT PNG_LIB_TARGETS)
  message(SEND_ERROR
    "No library variant selected to build. "
    "Please enable at least one of the following options: PNG_STATIC, PNG_SHARED")
endif()

if(PNG_SHARED AND WIN32)
  set_target_properties(${PNG_LIB_NAME} PROPERTIES DEFINE_SYMBOL PNG_BUILD_DLL)
endif()

if(PNG_TESTS AND PNG_SHARED AND NOT BIICODE)
  # does not work with msvc due to png_lib_ver issue
  add_executable(pngtest ${pngtest_sources})
  target_link_libraries(pngtest ${PNG_LIB_NAME})
  add_test(pngtest ./pngtest ${CMAKE_CURRENT_SOURCE_DIR}/pngtest.png)
  #
  add_executable(pngvalid ${pngvalid_sources})
  target_link_libraries(pngvalid ${PNG_LIB_NAME})
  add_test(pngvalid ./pngvalid)
  add_executable(pngstest ${pngstest_sources})
  target_link_libraries(pngstest ${PNG_LIB_NAME})
  add_test(pngstest ./pngstest
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn0g01.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn0g02.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn0g04.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn0g08.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn0g16.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn2c08.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn2c16.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn3p01.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn3p02.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn3p04.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn3p08.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn4a08.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn4a16.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn6a08.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/basn6a16.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftbbn0g01.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftbbn0g02.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftbbn0g04.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftbbn2c16.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftbbn3p08.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftbgn2c16.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftbgn3p08.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftbrn2c08.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftbwn0g16.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftbwn3p08.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftbyn3p08.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftp0n0g08.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftp0n2c08.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftp0n3p08.png
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/ftp1n3p08.png
  )
endif()

# Ensure the CMAKE_LIBRARY_OUTPUT_DIRECTORY is set
IF(NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)
  SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY "lib")
ENDIF(NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)

# libpng is a library so default to 'lib'
if(NOT DEFINED CMAKE_INSTALL_LIBDIR)
  set(CMAKE_INSTALL_LIBDIR lib)
endif(NOT DEFINED CMAKE_INSTALL_LIBDIR)

# SET UP LINKS
if(PNG_SHARED)
  set_target_properties(${PNG_LIB_NAME} PROPERTIES
#   VERSION 16.${PNGLIB_RELEASE}.1.6.15
    VERSION 16.${PNGLIB_RELEASE}.0
    SOVERSION 16
    CLEAN_DIRECT_OUTPUT 1)
endif()
if(PNG_STATIC)
  # MSVC doesn't use a different file extension for shared vs. static
  # libs.  We are able to change OUTPUT_NAME to remove the _static
  # for all other platforms.
  if(NOT MSVC)
    set_target_properties(${PNG_LIB_NAME_STATIC} PROPERTIES
      OUTPUT_NAME ${PNG_LIB_NAME}
      CLEAN_DIRECT_OUTPUT 1)
  endif()
endif()

# If CMake > 2.4.x, we set a variable used below to export
# targets to an export file.
# TODO: Use VERSION_GREATER after our cmake_minimum_required >= 2.6.2
if(CMAKE_MAJOR_VERSION GREATER 1 AND CMAKE_MINOR_VERSION GREATER 4)
  set(PNG_EXPORT_RULE EXPORT libpng)
elseif(CMAKE_MAJOR_VERSION GREATER 2) # future proof
  set(PNG_EXPORT_RULE EXPORT libpng)
endif()

