# cmake configuration

cmake_minimum_required(VERSION 2.8.12)
cmake_policy(VERSION 2.8.12)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH}
    "${CMAKE_SOURCE_DIR}/cmake/")

include(CheckIncludeFile)

# for /MT on MSVC
set(CMAKE_USER_MAKE_RULES_OVERRIDE
   "${CMAKE_SOURCE_DIR}/cmake/c_flag_overrides.cmake")
set(CMAKE_USER_MAKE_RULES_OVERRIDE_CXX
   "${CMAKE_SOURCE_DIR}/cmake/cxx_flag_overrides.cmake")

# project

project(solvespace)
set(solvespace_VERSION_MAJOR 2)
set(solvespace_VERSION_MINOR 1)

if(WIN32)
    # ttf2c takes 10 minutes (!) on Appveyor CI
    set(DISABLE_TTF2C CACHE BOOL "Disable font regeneration with ttf2c, for use on CI")
endif()

if(NOT WIN32)
    set(GUI gtk2 CACHE STRING "GUI toolkit to use (one of: gtk2 gtk3)")
endif()

# compiler

if(WIN32)
    add_definitions(
        -D_CRT_SECURE_NO_DEPRECATE=1
        -D_CRT_SECURE_NO_WARNINGS=1
        -D_WIN32_WINNT=0x500
        -D_WIN32_IE=_WIN32_WINNT
        -DISOLATION_AWARE_ENABLED=1
        -DWIN32=1
        -DWIN32_LEAN_AND_MEAN=1)
endif()

if((CMAKE_CXX_PLATFORM_ID STREQUAL "Linux") AND CMAKE_COMPILER_IS_GNUCC)
    set(CMAKE_EXE_LINKER_FLAGS "-Wl,--as-needed ${CMAKE_EXE_LINKER_FLAGS}")
endif()

if(MINGW)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -static-libgcc")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static-libgcc -static-libstdc++")
endif()

# dependencies

CHECK_INCLUDE_FILE("stdint.h" HAVE_STDINT_H)

find_package(OpenGL REQUIRED)

find_package(Perl)
find_package(PerlModules COMPONENTS GD)
if(NOT (PERL_FOUND AND PERLMODULES_FOUND))
    message(STATUS "Perl with GD not found; icons will not be regenerated if modified")
endif()

if(WIN32)
    find_package(PNG)

    if(NOT PNG_FOUND)
        message(STATUS "Using in-tree libpng")

        add_subdirectory(extlib/zlib)

        message(STATUS "Using in-tree libpng")

        set(ZLIB_LIBRARY
            zlibstatic)
        set(ZLIB_INCLUDE_DIR
            "${CMAKE_SOURCE_DIR}/extlib/zlib"
            "${CMAKE_BINARY_DIR}/extlib/zlib")
        set(SKIP_INSTALL_ALL
            ON)
        add_subdirectory(extlib/libpng)

        set(PNG_FOUND TRUE)
        set(PNG_LIBRARIES
            png16_static
            zlibstatic)
        set(PNG_INCLUDE_DIRS
            "${CMAKE_SOURCE_DIR}/extlib/libpng"
            "${CMAKE_BINARY_DIR}/extlib/libpng"
            "${CMAKE_SOURCE_DIR}/extlib/zlib"
            "${CMAKE_BINARY_DIR}/extlib/zlib")
    endif()

    if(NOT MINGW)
        message(STATUS "Using prebuilt SpaceWare")
        set(SPACEWARE_FOUND TRUE)
        set(SPACEWARE_INCLUDE_DIR
            "${CMAKE_SOURCE_DIR}/extlib/si")
        set(SPACEWARE_LIBRARIES
            "${CMAKE_SOURCE_DIR}/extlib/si/siapp.lib")
    endif()
elseif(APPLE)
    find_package(PNG REQUIRED)
    find_library(APPKIT_LIBRARY AppKit REQUIRED)
else() # Linux and compatible systems
    find_package(SpaceWare)

    # Use freedesktop's pkg-config to locate everything.
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(PNG REQUIRED libpng)
    pkg_check_modules(FONTCONFIG REQUIRED fontconfig)
    pkg_check_modules(JSONC REQUIRED json-c)
    pkg_check_modules(GLEW REQUIRED glew)

    set(HAVE_GTK TRUE)
    if(GUI STREQUAL "gtk3")
        set(HAVE_GTK3 TRUE)
        pkg_check_modules(GTKMM REQUIRED gtkmm-3.0 pangomm-1.4 x11)
    elseif(GUI STREQUAL "gtk2")
        set(HAVE_GTK2 TRUE)
        pkg_check_modules(GTKMM REQUIRED gtkmm-2.4 pangomm-1.4 x11)
    else()
        message(FATAL_ERROR "GUI unrecognized: ${GUI}")
    endif()
endif()



macro(set_config_option VARNAME STRING)
  set(${VARNAME} TRUE)
  list(APPEND CONFIG_OPTIONS ${STRING})
  message(STATUS "Found " ${STRING})
endmacro(set_config_option)


INCLUDE (CheckTypeSize)
check_type_size("void*" SIZEOF_VOID_P)
if(SIZEOF_VOID_P EQUAL 8)
  set_config_option(HAVE_64BIT_SIZE_T "Have64BitSizeT")
endif(SIZEOF_VOID_P EQUAL 8)

set(OCC_3PARTIES_LIBRARIES)
if(MSVC)
  set(OCC_3PARTIES_DLLS 
	"C:/occ3dparties/freeimage-3.16.0-vc12-64-opt/bin/FreeImage.dll"
	"C:/occ3dparties/freeimage-3.16.0-vc12-64-opt/bin/FreeImagePlus.dll"
        "C:/occ3dparties/freetype-2.5.3-vc12-64-opt/bin/freetype.dll"
	"C:/occ3dparties/gl2ps-1.3.8-vc12-64-opt/lib/gl2ps.dll"
  )
  set(OCC_3PARTIES_DLLS_DEB 
	"C:/occ3dparties/freeimage-3.16.0-vc12-64-deb/bin/FreeImage.dll"
	"C:/occ3dparties/freeimage-3.16.0-vc12-64-deb/bin/FreeImagePlus.dll"
        "C:/occ3dparties/freetype-2.5.3-vc12-64-opt/bin/freetype.dll"
	"C:/occ3dparties/gl2ps-1.3.8-vc12-64-deb/lib/gl2ps.dll"
  )
  set(OCC_Resources
    ${CASROOT}/src/StdResource/XCAF
    ${CASROOT}/src/StdResource/Plugin
  )
endif(MSVC)

if(MINGW)
  set(OCC_3PARTIES_LIBRARIES_REQUIRED
	"libFreeImage"
	"libFreeImagePlus"
        "libfreetype"
	"libgl2ps"
  )
  set(OCC_3PARTIES_DLLS)
  set(OCC_3PARTIES_DLLS_DEB)
  foreach(LIB_REQ ${OCC_3PARTIES_LIBRARIES_REQUIRED})
    set(DLL "DLL-NOTFOUND")
    find_file(DLL ${LIB_REQ}.dll)
    if(NOT ${DLL} STREQUAL "LIB-NOTFOUND")
       list(APPEND OCC_3PARTIES_DLLS ${DLL})
    else(NOT ${DLL} STREQUAL "LIB-NOTFOUND")
       message(SEND_ERROR ${LIB_REQ}.dll " not Found")
    endif(NOT ${DLL} STREQUAL "LIB-NOTFOUND")
    find_library(LIB ${LIB_REQ})
    if(NOT ${LIB} STREQUAL "LIB-NOTFOUND")
       list(APPEND OCC_3PARTIES_LIBRARIES ${LIB})
    else(NOT ${LIB} STREQUAL "LIB-NOTFOUND")
       message(SEND_ERROR "library ${LIB_REQ} not Found")
    endif(NOT ${LIB} STREQUAL "LIB-NOTFOUND")
    unset(DLL CACHE)
    unset(LIB CACHE)
  endforeach(LIB_REQ ${OCC_3PARTIES_LIBRARIES_REQUIRED})
  set(OCC_Resources
    ${CASROOT}/src/StdResource/XCAF
    ${CASROOT}/src/StdResource/Plugin
  )
endif(MINGW)


#Find OpenCascade
if(WIN32 OR MINGW)
    if(HAVE_64BIT_SIZE_T)
      set(OCC_SYS_NAME win64)
    else(HAVE_64BIT_SIZE_T)
      set(OCC_SYS_NAME win32)
    endif(HAVE_64BIT_SIZE_T)
  else(WIN32 OR MINGW)
    set(OCC_SYS_NAME ${CMAKE_SYSTEM_NAME})
  endif(WIN32 OR MINGW)
  set(OCC_LIBRARIES_REQUIRED
	   FWOSPlugin
	   PTKernel
	   TKBin
	   TKBinL
	   TKBinXCAF
	   TKBO
	   TKBool
	   TKBRep
	   TKCAF
	   TKCDF
	   TKernel
	   TKFeat
	   TKFillet
	   TKG2d
	   TKG3d
	   TKGeomAlgo
	   TKGeomBase
	   TKHLR
	   TKIGES
	   TKLCAF
	   TKMath
	   TKMesh
	   TKMeshVS
	   TKOffset
	   TKOpenGl
	   TKPCAF
	   TKPLCAF
	   TKPrim
	   TKPShape
	   TKService
           TKShapeSchema
  	   TKShHealing
	   TKStdLSchema
	   TKStdSchema
	   TKSTEP
           TKSTEP209
	   TKSTEPAttr
	   TKSTEPBase
	   TKSTL
	   TKTopAlgo
	   TKV3d
	   TKVRML
	   TKXCAF
	   TKXCAFSchema
	   TKXDEIGES
	   TKXDESTEP
	   TKXml
	   TKXmlL
	   TKXmlXCAF
	   TKXSBase
  )

  list(LENGTH OCC_LIBRARIES_REQUIRED NUM_OCC_LIBRARIES_REQUIRED)
  set(OCC_LIBRARIES)
  set(OCC_LIBRARIES_DEB)
  foreach(OCC ${OCC_LIBRARIES_REQUIRED})
    set(OCC_LIB "OCC_LIB-NOTFOUND")
    set(OCC_LIB_DEB "OCC_LIB_DEB-NOTFOUND")
    if(MSVC)
      find_library(OCC_LIB ${OCC} PATHS ${CASROOT}/${OCC_SYS_NAME}/vc12/lib  ${CASROOT}/${OCC_SYS_NAME}/vc10/lib NO_DEFAULT_PATH)
      find_library(OCC_LIB_DEB ${OCC} PATHS ${CASROOT}/${OCC_SYS_NAME}/vc12/libd  ${CASROOT}/${OCC_SYS_NAME}/vc10/libd NO_DEFAULT_PATH)
    elseif(MINGW)
      find_library(OCC_LIB ${OCC} PATHS ${OCE_INST_DIR}/${OCC_SYS_NAME} PATH_SUFFIXES lib bin)
    else(MSVC)
      find_library(OCC_LIB ${OCC} PATH_SUFFIXES oce/lib)
      if(${OCC_LIB} STREQUAL "OCC_LIB-NOTFOUND")
       find_library(OCC_LIB ${OCC} PATH_SUFFIXES lib) 
      endif()
    endif(MSVC) 
    if(NOT ${OCC_LIB} STREQUAL "OCC_LIB-NOTFOUND")
      list(APPEND OCC_LIBRARIES ${OCC_LIB})
      message(STATUS "Found  ${OCC} = ${OCC_LIB}")
    else(NOT ${OCC_LIB} STREQUAL "OCC_LIB-NOTFOUND")
      message(SEND_ERROR "${OCC} not Found")
    endif(NOT ${OCC_LIB} STREQUAL "OCC_LIB-NOTFOUND")
    if(MSVC)
    if(NOT ${OCC_LIB_DEB} STREQUAL "OCC_LIB_DEB-NOTFOUND")
      list(APPEND OCC_LIBRARIES_DEB ${OCC_LIB_DEB})
    else(NOT ${OCC_LIB_DEB} STREQUAL "OCC_LIB-NOTFOUND")
      list(APPEND OCC_LIBRARIES_DEB ${OCC_LIB})
      message(WARNING ${OCC} " not Found Debug Version")
    endif(NOT ${OCC_LIB_DEB} STREQUAL "OCC_LIB_DEB-NOTFOUND")
    unset(OCC_LIB_DEB CACHE) # only available in cmake >= 2.6.4
    endif(MSVC) 
    unset(OCC_LIB CACHE) # only available in cmake >= 2.6.4
  endforeach(OCC)

  if(WIN32)
   set(OCC_DLLS)
   set(OCC_DLLS_DEB)
   foreach(OCC ${OCC_LIBRARIES_REQUIRED})
    set(OCC_DLL "OCC_DLL-NOTFOUND")
    if(MSVC)
     find_file(OCC_DLL ${OCC}.dll PATHS ${CASROOT}/${OCC_SYS_NAME}/vc12/bin  ${CASROOT}/${OCC_SYS_NAME}/vc10/bin NO_DEFAULT_PATH)
    elseif(MINGW)
     find_file(OCC_DLL lib${OCC}.dll PATHS ${OCE_INST_DIR}/${OCC_SYS_NAME} PATH_SUFFIXES bin)
    endif()
    if(NOT ${OCC_DLL} STREQUAL "OCC_DLL-NOTFOUND")
      list(APPEND OCC_DLLS ${OCC_DLL})
    else(NOT ${OCC_DLL} STREQUAL "OCC_DLL-NOTFOUND")
      message(SEND_ERROR ${OCC} " not Found")
    endif(NOT ${OCC_DLL} STREQUAL "OCC_DLL-NOTFOUND")
    if(MSVC)
    set(OCC_DLL "OCC_DLL-NOTFOUND")
     find_file(OCC_DLL ${OCC}.dll PATHS ${CASROOT}/${OCC_SYS_NAME}/vc12/bind  ${CASROOT}/${OCC_SYS_NAME}/vc10/bind NO_DEFAULT_PATH)
     if(NOT ${OCC_DLL} STREQUAL "OCC_DLL-NOTFOUND")
      list(APPEND OCC_DLLS_DEB ${OCC_DLL})
     else(NOT ${OCC_DLL} STREQUAL "OCC_DLL-NOTFOUND")
      message(STATUS "Debug version of" ${OCC} " not Found")
     endif(NOT ${OCC_DLL} STREQUAL "OCC_DLL-NOTFOUND")
    endif(MSVC)
    unset(OCC_DLL CACHE) # only available in cmake >= 2.6.4
   endforeach(OCC ${OCC_LIBRARIES_REQUIRED})
  endif(WIN32)

  list(LENGTH OCC_LIBRARIES NUM_OCC_LIBRARIES)
  set(OCC_INCLUDES)
  if(NUM_OCC_LIBRARIES EQUAL NUM_OCC_LIBRARIES_REQUIRED)
  if(MINGW)
    find_path(OCC_INC "BRep_Tool.hxx" PATHS ${OCE_INST_DIR} PATH_SUFFIXES include/oce)
  elseif(MSVC)
    find_path(OCC_INC "BRep_Tool.hxx" PATHS ${CASROOT} PATH_SUFFIXES inc)
  else(MINGW)
    find_path(OCC_INC "BRep_Tool.hxx" PATH_SUFFIXES include/oce)
    if(${OCC_INC} STREQUAL "OCC_INC-NOTFOUND")
     find_path(OCC_INC "BRep_Tool.hxx" PATHS ${CASROOT} PATH_SUFFIXES inc)
    endif()
  endif(MINGW)
    if(OCC_INC)
      list(APPEND OCC_INCLUDES ${OCC_INC})
      set_config_option(HAVE_OCC "OpenCascade")
      ADD_DEFINITIONS(-DCSFDB)
      if(HAVE_64BIT_SIZE_T)
        add_definitions(-D_OCC67)
      endif(HAVE_64BIT_SIZE_T)
      find_path(OCC_CONFIG_H "config.h" PATHS ${OCC_INC} ${CASROOT} NO_DEFAULT_PATH)
      if(${OCC_CONFIG_H} STREQUAL "OCC_CONFIG_H-NOTFOUND")
        add_definitions(-DHAVE_NO_OCC_CONFIG_H)
      else(${OCC_CONFIG_H} STREQUAL "OCC_CONFIG_H-NOTFOUND")
        list(APPEND OCC_INCLUDES ${OCC_CONFIG_H})
	ADD_DEFINITIONS(-DHAVE_OCC_CONFIG_H)
      endif(${OCC_CONFIG_H} STREQUAL "OCC_CONFIG_H-NOTFOUND")
      find_path(OCE_CONFIG_H "oce-config.h" PATHS ${OCC_INC})
      if(${OCE_CONFIG_H} STREQUAL "OCE_CONFIG_H-NOTFOUND")
	add_definitions(-DHAVE_NO_OCE_CONFIG_H)
      else(${OCE_CONFIG_H} STREQUAL "OCE_CONFIG_H-NOTFOUND")
	ADD_DEFINITIONS(-DHAVE_OCE_CONFIG_H)
      endif(${OCE_CONFIG_H} STREQUAL "OCE_CONFIG_H-NOTFOUND")
    endif(OCC_INC)
  endif(NUM_OCC_LIBRARIES EQUAL NUM_OCC_LIBRARIES_REQUIRED)






# components

if(WIN32)
    add_subdirectory(tools)
endif()

add_subdirectory(src)
add_subdirectory(exposed)
