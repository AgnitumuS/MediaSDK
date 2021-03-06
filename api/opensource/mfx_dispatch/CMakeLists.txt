# Copyright (c) 2017 Intel Corporation
# 
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
# 
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
# 
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

cmake_minimum_required( VERSION 2.6.2 FATAL_ERROR )
project( mfx )

set( CMAKE_VERBOSE_MAKEFILE             TRUE )
set( CMAKE_LIB_DIR ${CMAKE_BINARY_DIR}/__lib )

if( ${CMAKE_SYSTEM_NAME} MATCHES Windows )
  add_definitions(-DMINGW_HAS_SECURE_API -DWIN32)

  if(__ARCH MATCHES intel64)
    add_definitions(-DWIN64)
  endif( )
else( )
  add_definitions(-DUNIX)

  if( ${CMAKE_SYSTEM_NAME} MATCHES Linux )
    add_definitions(-D__USE_LARGEFILE64 -D_FILE_OFFSET_BITS=64 -DLINUX -DLINUX32)

    if(__ARCH MATCHES intel64)
      add_definitions(-DLINUX64)
    endif( )
  endif( )

  if( ${CMAKE_SYSTEM_NAME} MATCHES Darwin )
    add_definitions(-DOSX)
    add_definitions(-DOSX32)

    if(__ARCH MATCHES intel64)
      add_definitions(-DOSX64)
    endif( )
  endif( )

  set(no_warnings "-Wno-unknown-pragmas -Wno-unused")

  set(CMAKE_CXX_FLAGS "-pipe -fPIC")
  set(CMAKE_CXX_FLAGS_DEBUG   "-O0 -Wall ${no_warnings} -g -D_DEBUG" CACHE STRING "" FORCE)
  set(CMAKE_CXX_FLAGS_RELEASE "-O2 -D_FORTIFY_SOURCE=2 -fstack-protector -Wall ${no_warnings} -DNDEBUG"    CACHE STRING "" FORCE)

  if (DEFINED CMAKE_FIND_ROOT_PATH)
    append("--sysroot=${CMAKE_FIND_ROOT_PATH} " LINK_FLAGS)
  endif (DEFINED CMAKE_FIND_ROOT_PATH)
endif( )

include_directories (
  ${CMAKE_CURRENT_SOURCE_DIR}/../../include
  ${CMAKE_CURRENT_SOURCE_DIR}/include
)

list(APPEND sources
  ${CMAKE_CURRENT_SOURCE_DIR}/src/main.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_critical_section.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_critical_section_linux.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_dispatcher.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_function_table.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_library_iterator.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_library_iterator_linux.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_load_dll.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_load_dll_linux.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_win_reg_key.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_dxva2_device.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_plugin_hive.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_plugin_hive_linux.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_plugin_cfg_parser.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_load_plugin.cpp
  )

function( append_property target property_name property )
  get_target_property( property ${ARGV0} ${ARGV1} )
  if( property MATCHES NOTFOUND)
    set( property "" )
  endif( )
  string( REPLACE ";" " " property "${ARGV2} ${property}" )
  set_target_properties( ${ARGV0} PROPERTIES ${ARGV1} "${property}" )
endfunction( )

function( make_static_library name )
  if( sources.plus )
    list( APPEND sources ${sources.plus} )
  endif( )

  set( target ${ARGV0} )

  add_library( ${target} STATIC ${include} ${sources} )

  set( link_flags_list "-Wl,--no-undefined,-z,relro,-z,now,-z,noexecstack")
  append_property( ${target} LINK_FLAGS "${link_flags_list} -fstack-protector" )

  if( defs )
    append_property( ${target} COMPILE_FLAGS ${defs} )
  endif( )

  set_target_properties( ${target} PROPERTIES ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_LIB_DIR}/${CMAKE_BUILD_TYPE} FOLDER ${target} )
endfunction( )

set( defs "${WARNING_FLAGS}" )
make_static_library( mfx )

set( defs "-DMFX_DISPATCHER_EXPOSED_PREFIX ${WARNING_FLAGS}" )
make_static_library( dispatch_shared )

list(APPEND sources.plus
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mfx_dispatcher_log.cpp
  )

set( defs "-DMFX_DISPATCHER_LOG -DDXVA2DEVICE_LOG ${WARNING_FLAGS}" )
make_static_library( dispatch_trace )

get_mfx_version(MFX_VERSION_MAJOR MFX_VERSION_MINOR)
set( PKG_CONFIG_FNAME "${CMAKE_LIB_DIR}/${CMAKE_BUILD_TYPE}/${PROJECT_NAME}.pc")

set( INSTALL_LIB_PATH "lib/lin_x86" )
if(__ARCH MATCHES intel64)
  set( INSTALL_LIB_PATH "lib/lin_x64" )
endif( )

configure_file("${CMAKE_CURRENT_SOURCE_DIR}/pkg-config.pc.cmake" ${PKG_CONFIG_FNAME} @ONLY)

install( TARGETS mfx ARCHIVE DESTINATION ${INSTALL_LIB_PATH} )
install( TARGETS dispatch_shared ARCHIVE DESTINATION ${INSTALL_LIB_PATH} )
install( TARGETS dispatch_trace ARCHIVE DESTINATION ${INSTALL_LIB_PATH} )
install( FILES ${PKG_CONFIG_FNAME} DESTINATION "${INSTALL_LIB_PATH}/pkgconfig" )

install( DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/../../include/ DESTINATION include FILES_MATCHING PATTERN *.h )
