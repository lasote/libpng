# Initializes block variables
INIT_BIICODE_BLOCK()

option(PNG_TESTS  "Build libpng tests" ON)
option(PNG_DEBUG  "Build with debug output" NO)
option(PNGARG     "Disable ANSI-C prototypes" NO)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scripts/pnglibconf.h.prebuilt
               ${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.h)

include_directories(${CMAKE_CURRENT_BINARY_DIR})

# OUR SOURCES
set(libpng_public_hdrs
  png.h
  pngconf.h
  ${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.h
)
set(BII_LIB_SRC
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

# Excluded examples (TODO: FIX COMPILATION WITH BIICODE)
LIST(REMOVE_ITEM BII_BLOCK_EXES contrib_gregbook_rpng-x)
LIST(REMOVE_ITEM BII_BLOCK_EXES contrib_gregbook_rpng2-x)
LIST(REMOVE_ITEM BII_BLOCK_EXES contrib_gregbook_wpng)
LIST(REMOVE_ITEM BII_BLOCK_EXES contrib_libtests_tarith)
 
IF(WIN32)
	LIST(REMOVE_ITEM BII_BLOCK_EXES contrib_libtests_timepng)
ELSE()

ENDIF()

ADD_BIICODE_TARGETS()
# ADD BINARY DIR TO FIND THE CONFIGURED 
TARGET_INCLUDE_DIRECTORIES(${BII_BLOCK_TARGET} INTERFACE ${CMAKE_CURRENT_BINARY_DIR})
if(MSVC)
TARGET_COMPILE_OPTIONS(${BII_BLOCK_TARGET} INTERFACE "-D_CRT_SECURE_NO_DEPRECATE")
endif(MSVC)

##biicode translation
IF(NOT WIN32)
  target_link_libraries(${BII_BLOCK_TARGET} INTERFACE m)
ENDIF()
