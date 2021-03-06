# Copyright: (C) 2009 RobotCub Consortium
# Authors: Paul Fitzpatrick, Arjan Gijsberts, Lorenzo Natale, Fabien Benureau, Stephane Lallee, Ali Paikan
# CopyPolicy: Released under the terms of the LGPLv2.1 or later, see LGPL.TXT

cmake_minimum_required(VERSION 2.8.9)

include(CMakeDependentOption)
include(CheckCXXCompilerFlag)
include(GNUInstallDirs)

set(YARP_COMPILE_BINDINGS_DEFAULT TRUE)
set(YARP_COLLISION_AVOIDANCE FALSE)
set(YARP_BASE_DIR ${CMAKE_CURRENT_BINARY_DIR})
if(YARP_VERSION)
  # this is a nested build
  set(YARP_COMPILE_BINDINGS_DEFAULT FALSE)
  set(YARP_COLLISION_AVOIDANCE TRUE)
  set(YARP_BASE_DIR ${CMAKE_BINARY_DIR})
endif()
option(YARP_COMPILE_BINDINGS "Compile optional language bindings" ${YARP_COMPILE_BINDINGS_DEFAULT})

#############################################################################
## Options for compiling supported languages.  There's nothing magical
## about this list, any SWIG-supported language should work - take a
## look at e.g. ruby code below for how to do it.

foreach(lang "Java"
             "Python"
             "Perl"
             "Chicken Scheme"
             "CSharp"
             "Allegro Common Lisp"
             "TCL"
             "Ruby"
             "Lua"
             "Octave")
  string(TOUPPER "${lang}" LANG)
  string(REGEX REPLACE " .+" "" LANG "${LANG}")
  cmake_dependent_option(CREATE_${LANG} "Do you want to create the ${lang} interface" OFF
                         YARP_COMPILE_BINDINGS OFF)
  if(YARP_COMPILE_BINDINGS)
    if(CREATE_${LANG})
      message(STATUS " +++ bindings ${lang} is enabled")
    else()
      message(STATUS " +++ bindings ${lang} is not enabled")
    endif()
  endif()
endforeach()


if(YARP_COMPILE_BINDINGS)

#############################################################################
## Do not build deprecated functions when disabled

if(YARP_NO_DEPRECATED)
  set(SWIG_COMMON_FLAGS "-DYARP_NO_DEPRECATED")
endif()


#############################################################################
## Discourage inplace compilation

if(CMAKE_BINARY_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
  option(COMPILE_INPLACE "Allow inplace compilation" FALSE)
  if(NOT COMPILE_INPLACE)
    if(NOT BEND_OVER_BACKWARDS_COMPATIBLE)
      message(FATAL_ERROR "Please don't compile bindings in the source directory, make a separate build directory (ideally one per language).  If CMake has now added a CMakeCache.txt file in the source directory, please delete it.  If you really really want to compile in place, set the COMPILE_INPLACE flag.")
    endif()
  endif()
endif()

#############################################################################
## Find SWIG and YARP

if (YARP_SPECIAL_SWIG)
  set(__CMAKE_IGNORE_PATH ${CMAKE_IGNORE_PATH})
  set(CMAKE_IGNORE_PATH /usr/bin ${CMAKE_IGNORE_PATH})
  find_package(SWIG REQUIRED)
  set(CMAKE_IGNORE_PATH ${__CMAKE_IGNORE_PATH})
else()
  find_package(SWIG REQUIRED)
endif()
include(${SWIG_USE_FILE})
message(STATUS "${SWIG_USE_FILE}")
set(MIN_SWIG_VERSION "1.3.29")
if("${SWIG_VERSION}" VERSION_LESS "${MIN_SWIG_VERSION}")
  message(FATAL_ERROR "Please update your SWIG. Current version: ${SWIG_VERSION}. The minimum version needed: ${MIN_SWIG_VERSION}")
endif()

if(NOT YARP_VERSION)
  find_package(YARP REQUIRED)
  list(REMOVE_ITEM YARP_LIBRARIES YARP::YARP_math)
else()
  set(YARP_FOUND TRUE)
  get_property(YARP_INCLUDE_DIRS GLOBAL PROPERTY YARP_TREE_INCLUDE_DIRS)
  get_property(YARP_LIBRARIES GLOBAL PROPERTY YARP_LIBS)
  list(REMOVE_ITEM YARP_LIBRARIES YARP_math)
  get_property(YARP_DEFINES GLOBAL PROPERTY YARP_DEFS)
endif()
include_directories(${YARP_INCLUDE_DIRS})
link_libraries(${YARP_LIBRARIES})

set_source_files_properties(yarp.i PROPERTIES CPLUSPLUS ON)


#############################################################################
## Disable deprecated warnings, since we are building bindings also for
## deprecated methods

if(NOT YARP_NO_DEPRECATED)
  check_cxx_compiler_flag("-Wno-deprecated-declarations" CXX_HAS_WNO_DEPRECATED_DECLARATIONS)
  if(CXX_HAS_WNO_DEPRECATED_DECLARATIONS)
    add_definitions(-Wno-deprecated-declarations)
  endif()
endif()


#############################################################################
## Create Java bindings

if(CREATE_JAVA)
  find_package(JNI REQUIRED)
  find_program(JAVA_JAVAC_EXECUTABLE NAMES javac)

  option(PREPARE_CLASS_FILES "Prepare class files" OFF)
  set(JAVA_TARGET_VERSION 1.6 CACHE STRING "Java target version")
  set(CMAKE_SWIG_OUTDIR_SAVE ${CMAKE_SWIG_OUTDIR})
  set(CMAKE_SWIG_OUTDIR "${YARP_BASE_DIR}/generated_src/java")
  set(CMAKE_SWIG_FLAGS "-package;yarp;-Wall;-module;yarp;${SWIG_COMMON_FLAGS}")
  set(TARGET_DIR generated_src/java)
  file(MAKE_DIRECTORY "${YARP_BASE_DIR}/${TARGET_DIR}")
  if(NOT BINDINGS_SRC_DIR)
    set(BINDINGS_SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src)
  endif()
  set(JAVA_SRCS LoadYarp.java
                YarpImageHelper.java
                Example1.java)
  include_directories(SYSTEM ${JAVA_INCLUDE_PATH} ${JAVA_INCLUDE_PATH2})
  set(target_name jyarp)
  swig_add_module(${target_name} java yarp.i)
  if(PREPARE_CLASS_FILES)
    set(CLASS_FILES)
    foreach(java_file ${JAVA_SRCS})
      string(REPLACE ".java" ".class" class_file ${java_file})
      list(APPEND JAVA_FILES "${BINDINGS_SRC_DIR}/${java_file}")
      list(APPEND CLASS_FILES "${YARP_BASE_DIR}/${class_file}")
    endforeach()
    add_custom_command(OUTPUT ${CLASS_FILES}
                       COMMAND ${JAVA_JAVAC_EXECUTABLE} -source ${JAVA_TARGET_VERSION}
                                                        -target ${JAVA_TARGET_VERSION}
                                                        -d "${YARP_BASE_DIR}"
                                                        "${YARP_BASE_DIR}/${TARGET_DIR}/*.java"
                       COMMAND ${JAVA_JAVAC_EXECUTABLE} -source ${JAVA_TARGET_VERSION}
                                                        -target ${JAVA_TARGET_VERSION}
                                                        -d "${YARP_BASE_DIR}/"
                                                        "${BINDINGS_SRC_DIR}/*.java"
                       WORKING_DIRECTORY ${YARP_BASE_DIR}
                       DEPENDS ${JAVA_FILES} ${target_name})
    add_custom_target(${target_name}_classes ALL DEPENDS ${CLASS_FILES})
  endif()

  # issue on MINGW where C++ name-mangling prevents java finding methods
  if(MINGW)
    message(STATUS "untested MINGW patch - see CMakeLists.txt")
    set_target_properties(${target_name} PROPERTIES
      LINK_FLAGS "--add-stdcall-alias")
  endif(MINGW)
  ## the following is based on:
  ## http://www.cmake.org/pipermail/cmake/2007-September/016343.html
  if(APPLE)
    set_target_properties(${target_name} PROPERTIES SUFFIX ".jnilib")
  endif(APPLE)
  set(YARP_COLLISION_AVOIDANCE TRUE)
  set(CMAKE_SWIG_OUTDIR ${CMAKE_SWIG_OUTDIR_SAVE})
endif(CREATE_JAVA)



#############################################################################
## Create Python bindings

if(CREATE_PYTHON)
  set(CMAKE_SWIG_FLAGS "-Wall;-module;yarp;-threads;${SWIG_COMMON_FLAGS}")
  if(CREATE_PYTHON_VERSION)
    message(WARNING "CREATE_PYTHON_VERSION was renamed YARP_USE_PYTHON_VERSION")
    set(YARP_USE_PYTHON_VERSION "${CREATE_PYTHON_VERSION}" CACHE STRING "Specify python version to use" )
    unset(CREATE_PYTHON_VERSION CACHE)
  else()
    set(YARP_USE_PYTHON_VERSION "" CACHE STRING "Specify python version to use" )
  endif()
  find_package(PythonInterp ${YARP_USE_PYTHON_VERSION} REQUIRED)
  set(YARP_USE_PYTHON_VERSION_ARGS)
  if(NOT YARP_USE_PYTHON_VERSION)
    set (YARP_USE_PYTHON_VERSION ${PYTHON_VERSION_STRING})
  endif()
  find_package(PythonLibs ${YARP_USE_PYTHON_VERSION} EXACT)
  include_directories(SYSTEM ${PYTHON_INCLUDE_PATH})

  ## in Windows it seems necessary to declare explictly the link_directory
  if(WIN32)
    get_filename_component(PYTHON_DIR ${PYTHON_LIBRARY} PATH)
    link_directories(${PYTHON_DIR})
  endif()

  set(target_name yarp)
  if(YARP_COLLISION_AVOIDANCE)
    set(target_name yarp_python)
    set(CMAKE_SWIG_OUTDIR_SAVE ${CMAKE_SWIG_OUTDIR})
    set(CMAKE_SWIG_OUTDIR "${CMAKE_BINARY_DIR}/lib/python")
  endif()
  swig_add_module(${target_name} python yarp.i)
  target_link_libraries(_${target_name} ${PYTHON_LIBRARY})
  # uncomment if we want a debug version of the library
  #add_definitions(-DPy_DEBUG)

  set_target_properties(_${target_name} PROPERTIES OUTPUT_NAME "_yarp")
  if(YARP_COLLISION_AVOIDANCE)
    set_target_properties(_${target_name} PROPERTIES LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib/python")
  endif()

  # installation path is determined reliably on most platforms using distutils
  execute_process(COMMAND ${PYTHON_EXECUTABLE} -c "from distutils import sysconfig; print(sysconfig.get_python_lib(1,0,prefix=''))"
                  OUTPUT_VARIABLE PYTHON_INSTDIR
                  OUTPUT_STRIP_TRAILING_WHITESPACE )
  if(YARP_COLLISION_AVOIDANCE)
    install(FILES ${CMAKE_BINARY_DIR}/lib/python/yarp.py
            DESTINATION ${PYTHON_INSTDIR})
  else()
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/yarp.py
            DESTINATION ${PYTHON_INSTDIR})
  endif()
  install(TARGETS _${target_name}
          DESTINATION ${PYTHON_INSTDIR})

  if(WIN32)
    set_target_properties(_${target_name} PROPERTIES SUFFIX ".pyd")
  endif(WIN32)

  if(YARP_COLLISION_AVOIDANCE)
    set(CMAKE_SWIG_OUTDIR ${CMAKE_SWIG_OUTDIR_SAVE})
  else()
    set(YARP_COLLISION_AVOIDANCE TRUE)
  endif()
endif(CREATE_PYTHON)



#############################################################################
## Create Perl bindings

if(CREATE_PERL)
  set(CMAKE_SWIG_FLAGS "-Wall;-module;yarp;${SWIG_COMMON_FLAGS}")
  find_package(PerlLibs)
  include_directories(SYSTEM ${PERL_INCLUDE_PATH})

  get_filename_component(_CMAKE_INSTALL_PERLDIR ${PERL_LIBRARY} PATH)
  file(RELATIVE_PATH _CMAKE_INSTALL_PERLDIR "${_CMAKE_INSTALL_PERLDIR}" "${PERL_VENDORARCH}")
  set(_CMAKE_INSTALL_PERLDIR "${CMAKE_INSTALL_LIBDIR}/${_CMAKE_INSTALL_PERLDIR}")
  set(CMAKE_INSTALL_PERLDIR "${_CMAKE_INSTALL_PERLDIR}" CACHE PATH "perl bindings (${_CMAKE_INSTALL_PERLDIR})")
  mark_as_advanced(CMAKE_INSTALL_PERLDIR)
  if(NOT IS_ABSOLUTE ${CMAKE_INSTALL_PERLDIR})
    set(CMAKE_INSTALL_FULL_PERLDIR "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_PERLDIR}")
  else()
    set(CMAKE_INSTALL_FULL_PERLDIR "${CMAKE_INSTALL_PERLDIR}")
  endif()

  set(target_name yarp)
  if(YARP_COLLISION_AVOIDANCE)
    set(target_name yarp_perl)
    set(CMAKE_SWIG_OUTDIR_SAVE ${CMAKE_SWIG_OUTDIR})
    set(CMAKE_SWIG_OUTDIR "${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_PERLDIR}")
  endif()

  swig_add_module(${target_name} perl5 yarp.i)
  set_target_properties(${target_name} PROPERTIES PREFIX "")
  set_target_properties(${target_name} PROPERTIES OUTPUT_NAME "yarp")
  if(YARP_COLLISION_AVOIDANCE)
    set_target_properties(${target_name} PROPERTIES LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_PERLDIR}/auto/yarp")
  endif()

  # Update RPATH
  if(NOT CMAKE_SKIP_RPATH AND NOT CMAKE_SKIP_INSTALL_RPATH)
    file(RELATIVE_PATH _rel_path "${CMAKE_INSTALL_FULL_PERLDIR}" "${CMAKE_INSTALL_FULL_LIBDIR}")
    get_target_property(_current_rpath "${target_name}" INSTALL_RPATH)
    if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
      list(APPEND _current_rpath "@loader_path/${_rel_path}")
    else()
      list(APPEND _current_rpath "\$ORIGIN/${_rel_path}")
    endif()
    set_target_properties("${target_name}" PROPERTIES INSTALL_RPATH "${_current_rpath}")
  endif()

  install(TARGETS ${target_name}
          DESTINATION ${CMAKE_INSTALL_PERLDIR}/auto/yarp)
  install(FILES ${CMAKE_SWIG_OUTDIR}/yarp.pm
          DESTINATION ${CMAKE_INSTALL_PERLDIR})

  if(YARP_COLLISION_AVOIDANCE)
    set(CMAKE_SWIG_OUTDIR ${CMAKE_SWIG_OUTDIR_SAVE})
  else()
    set(YARP_COLLISION_AVOIDANCE TRUE)
  endif()
endif(CREATE_PERL)




#############################################################################
## Create Chicken bindings

if(CREATE_CHICKEN)
  set(CMAKE_SWIG_FLAGS "-Wall;-module;cyarp;${SWIG_COMMON_FLAGS}")
  swig_add_module(libcyarp chicken yarp.i)
  set(YARP_COLLISION_AVOIDANCE TRUE)
endif(CREATE_CHICKEN)



#############################################################################
## Create Allegro bindings

if(CREATE_ALLEGRO)
  set(CMAKE_SWIG_OUTDIR "${CMAKE_CURRENT_BINARY_DIR}")
  set(ALLEGRO_INSTALL_DIR "lib/allegro/site" CACHE PATH "Install path of allegro bindings.")
  set(ALLEGRO_ASDF_SYSTEMS_DIR "lib/allegro/asdf-systems" CACHE PATH "Path to register asdf file.")

  set(allegro_PROJ_FILE allegro/yarp.asd)
  set(allegro_SRC_FILE yarp.cl)
  set(allegro_LIBS yarp-allegro.so)

  set(CMAKE_SWIG_FLAGS "-Wall;-module;yarp;-identifier-converter;lispify;${SWIG_COMMON_FLAGS}")
  swig_add_module(yarp-allegro allegrocl yarp.i)

  string(REGEX REPLACE "\\.cl" ".lisp" allegro_DEST_FILE ${allegro_SRC_FILE})
  install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/${allegro_PROJ_FILE}
          DESTINATION ${ALLEGRO_INSTALL_DIR}/yarp)
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${allegro_SRC_FILE}
          DESTINATION ${ALLEGRO_INSTALL_DIR}/yarp/src
          RENAME ${allegro_DEST_FILE})
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${allegro_LIBS}
          DESTINATION ${ALLEGRO_INSTALL_DIR}/yarp/lib)

  if(UNIX)
    install(CODE "execute_process(COMMAND \"\${CMAKE_COMMAND}\" \"-E\" \"make_directory\" \"${ALLEGRO_ASDF_SYSTEMS}\"
                                  COMMAND \"ln\" \"-s\" \"${CMAKE_INSTALL_PREFIX}/${ALLEGRO_INSTALL_DIR}/yarp/yarp.asd\" \"${CMAKE_INSTALL_PREFIX}/${ALLEGRO_ASDF_SYSTEMS}\")")
  endif()
endif()

if(CREATE_TCL)
  set(CMAKE_SWIG_FLAGS "-Wall;-module;yarp;${SWIG_COMMON_FLAGS}")
  find_package(TCL REQUIRED)
  include_directories(SYSTEM ${TCL_INCLUDE_PATH})
  link_libraries(${TCL_LIBRARY})
  swig_add_module(libtclyarp tcl yarp.i)
  set(YARP_COLLISION_AVOIDANCE TRUE)
endif(CREATE_TCL)



#############################################################################
## Create C# bindings

if(CREATE_CSHARP)

  # outdir does not seem to work?
  set(target_name yarp)
  if(YARP_COLLISION_AVOIDANCE)
    set(target_name yarp_cs)
  endif()

  if(NOT ${CMAKE_MINIMUM_REQUIRED_VERSION} VERSION_LESS 3.0)
    message(AUTHOR_WARNING "CMAKE_MINIMUM_REQUIRED_VERSION is now ${CMAKE_MINIMUM_REQUIRED_VERSION}. This check can be removed.")
  endif()
  # -dllimport is actually needed only for CMake versions prior to 3.0 ( http://public.kitware.com/Bug/view.php?id=13814 )
  if(${CMAKE_VERSION} VERSION_LESS 3.0)
    set(CMAKE_SWIG_FLAGS "-Wall;-dllimport;${target_name};${SWIG_COMMON_FLAGS}")
  else()
    set(CMAKE_SWIG_FLAGS "-Wall;${SWIG_COMMON_FLAGS}")
  endif()

  if(WIN32 AND NOT CYGWIN)
    option(PREPARE_CLASS_FILES "Prepare NET library" ON)
    swig_add_module(${target_name} csharp yarp.i)
    if(PREPARE_CLASS_FILES)
      add_custom_command(TARGET ${target_name}
                         POST_BUILD
                         COMMAND IF NOT EXIST .\\generated_src mkdir generated_src
                         WORKING_DIRECTORY ${YARP_BASE_DIR})

      add_custom_command(TARGET ${target_name}
                         POST_BUILD
                         COMMAND move *.cs ${YARP_BASE_DIR}/generated_src
                         WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
    endif()
  else(WIN32 AND NOT CYGWIN)
    swig_add_module(${target_name} csharp yarp.i)
  endif(WIN32 AND NOT CYGWIN)
  set(YARP_COLLISION_AVOIDANCE TRUE)
endif(CREATE_CSHARP)



#############################################################################
## Create Ruby bindings

if(CREATE_RUBY)
  find_package(Ruby)
  set(CMAKE_SWIG_FLAGS "-Wall;-module;yarp;${SWIG_COMMON_FLAGS}")
  include_directories(SYSTEM ${RUBY_INCLUDE_PATH})
  set(target_name yarp)
  if(YARP_COLLISION_AVOIDANCE)
    set(target_name yarp_ruby)
  endif()
  swig_add_module(${target_name} ruby yarp.i)
  swig_link_libraries(${target_name} ${RUBY_LIBRARY})
  set_target_properties(${target_name} PROPERTIES PREFIX "")
  set_target_properties(${target_name} PROPERTIES OUTPUT_NAME "yarp")
  set(YARP_COLLISION_AVOIDANCE TRUE)
endif(CREATE_RUBY)




#############################################################################
## Create Lua bindings

if(CREATE_LUA)
  set(CMAKE_SWIG_FLAGS "-Wall;-module;yarp;${SWIG_COMMON_FLAGS}")
  find_package(Lua REQUIRED)
  set(target_name yarp_lua)
  include_directories(SYSTEM ${LUA_INCLUDE_DIR})
  swig_add_module(${target_name} lua yarp.i)
  swig_link_libraries(${target_name} ${LUA_LIBRARY})
  set_target_properties(${target_name} PROPERTIES PREFIX "")
  set_target_properties(${target_name} PROPERTIES OUTPUT_NAME "yarp")
  if(YARP_COLLISION_AVOIDANCE)
    set_target_properties(${target_name} PROPERTIES LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib/lua)
  endif()
  set(YARP_COLLISION_AVOIDANCE TRUE)

  set(_CMAKE_INSTALL_LUADIR "lib/lua/${LUA_VERSION_MAJOR}.${LUA_VERSION_MINOR}")
  set(CMAKE_INSTALL_LUADIR ${_CMAKE_INSTALL_LUADIR} CACHE PATH "lua bindings (${_CMAKE_INSTALL_LUADIR})")
  mark_as_advanced(CMAKE_INSTALL_LUADIR)
  if(NOT IS_ABSOLUTE ${CMAKE_INSTALL_LUADIR})
    set(CMAKE_INSTALL_FULL_LUADIR "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LUADIR}")
  else()
    set(CMAKE_INSTALL_FULL_LUADIR "${CMAKE_INSTALL_LUADIR}")
  endif()

  # Update RPATH
  if(NOT CMAKE_SKIP_RPATH AND NOT CMAKE_SKIP_INSTALL_RPATH)
    file(RELATIVE_PATH _rel_path "${CMAKE_INSTALL_FULL_LUADIR}" "${CMAKE_INSTALL_FULL_LIBDIR}")
    get_target_property(_current_rpath "${target_name}" INSTALL_RPATH)
    if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
      list(APPEND _current_rpath "@loader_path/${_rel_path}")
    else()
      list(APPEND _current_rpath "\$ORIGIN/${_rel_path}")
    endif()
    set_target_properties("${target_name}" PROPERTIES INSTALL_RPATH "${_current_rpath}")
  endif()

  install(TARGETS ${target_name}
          DESTINATION ${CMAKE_INSTALL_LUADIR})

endif(CREATE_LUA)

#############################################################################
## Create Octave bindings

if(CREATE_OCTAVE)
  set(CMAKE_SWIG_FLAGS "-Wall;-module;yarp;${SWIG_COMMON_FLAGS}")
  find_package(Octave REQUIRED)
  set(target_name yarp_octave)
  get_filename_component(OCTAVE_INCLUDE_TOP ${OCTAVE_INCLUDE_DIR} DIRECTORY)
  include_directories(SYSTEM ${OCTAVE_INCLUDE_DIRS} ${OCTAVE_INCLUDE_TOP})
  swig_add_module(${target_name} octave yarp.i)
  swig_link_libraries(${target_name} ${OCTAVE_LIBRARIES})
  set_target_properties(${target_name} PROPERTIES PREFIX "")
  set_target_properties(${target_name} PROPERTIES OUTPUT_NAME "yarp")
  set_target_properties(${target_name} PROPERTIES SUFFIX ".oct")
  if(YARP_COLLISION_AVOIDANCE)
    set_target_properties(${target_name} PROPERTIES LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib/octave)
  endif()
  set(YARP_COLLISION_AVOIDANCE TRUE)
endif(CREATE_OCTAVE)

endif()
