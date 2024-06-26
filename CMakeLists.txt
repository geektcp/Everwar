# build server

project(UeCore)
set(EVERWAR_VERSION 1.3)
cmake_minimum_required(VERSION 3.0)

include(cmake/common.cmake)

# Set RPATH-handing (CMake parameters)
set(CMAKE_SKIP_BUILD_RPATH FALSE)
set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
set(CMAKE_INSTALL_RPATH ${LIBS_DIR})
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

set(CMAKE_CXX_STANDARD 11)

set(EVERWAR_BASE /usr/local)

# set default buildoptions and print them
include(cmake/options.cmake)

set(EVERWAR_INSTALL UeCore)
set(CMAKE_INSTALL_PREFIX ${EVERWAR_BASE}/${EVERWAR_INSTALL}-${EVERWAR_VERSION})

set(BOOST_BASE boost)
set(BOOST_ROOT ${EVERWAR_BASE}/${BOOST_BASE})
link_directories(${EVERWAR_BASE}/${BOOST_BASE}/lib)
include_directories(${EVERWAR_BASE}/${BOOST_BASE}/include)

# Force out-of-source build
if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
  message(FATAL_ERROR
    "This project requires an out of source build. Remove the file 'CMakeCache.txt' found in this directory before continuing, create a separate build directory and run 'cmake [options] <srcs>' from there."
  )
endif()

# TODO: allow other compilers under windows in the future
if(WIN32 AND NOT MSVC)
  message(FATAL_ERROR
    "Under Windows other compiler than Microsoft Visual Studio are not supported."
  )
endif()

# TODO: remove this in the future! it has only been added to make the switch easier for end users
if(PREFIX)
  message(FATAL_ERROR "The parameter PREFIX has been removed. Please re-run CMake and use CMAKE_INSTALL_PREFIX instead to define your installation location!")
endif()


option(ENABLE_WARNINGS_SETTINGS "Allow target_set_warnings to add flags and defines.
                                 Set this to OFF if you want to provide your own warning parameters." ON)
option(ENABLE_LTO "Enable link time optimization" ON)

include(CheckPlatform)
include(Warnings)
find_package(MySQL)
include(Colors)
include(LTO)
include(Warnings)
include(AutoCollect)

# Check for LTO support.
find_lto(CXX)

list(APPEND CMAKE_MODULE_PATH
        ${CMAKE_SOURCE_DIR}/cmake/macros)



# TODO: use MSVC_CXX_ARCHITECTURE_ID instead to identify platform on windows (not required on other systems)
# find platform: required to build 3rd party libraries w/o CMake files
# Find out what system we use to include the needed libs
if(WIN32)
  if(PLATFORM MATCHES X86) # 32-bit
    set(DEP_ARCH win32)
  else() # 64-bit
    set(DEP_ARCH x64)
  endif()
endif()

# find Git: used to get the revision number
find_package(Git)

# check if the platform supports precomiled headers
#find_package(PCHSupport)
include(cotire)

# Override configuration-types - we don't use anything else than debug and release
if(CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_CONFIGURATION_TYPES Release Debug)
  set(CMAKE_CONFIGURATION_TYPES "${CMAKE_CONFIGURATION_TYPES}" CACHE STRING
    "Reset the configurations to what we need"
    FORCE)
endif()

set(BIN_FOLDER_NAME bin)
set(CONF_FOLDER_NAME etc)
set(LIBS_FOLDER_NAME lib)

set(BIN_DIR ${CMAKE_INSTALL_PREFIX}/${BIN_FOLDER_NAME})
set(LIB_DIR ${CMAKE_INSTALL_PREFIX}/${LIBS_FOLDER_NAME})
set(CONF_DIR ${CMAKE_INSTALL_PREFIX}/${CONF_FOLDER_NAME})
# If win32 put it in the bin dir not lib
if(WIN32)
  set(LIBS_DIR ${CMAKE_INSTALL_PREFIX}/${BIN_FOLDER_NAME})
else()
  set(LIBS_DIR ${CMAKE_INSTALL_PREFIX}/${LIBS_FOLDER_NAME})
endif()

# For Unix systems set the rpath so that libraries are found
set(CMAKE_INSTALL_RPATH ../${LIBS_FOLDER_NAME})
set(CMAKE_INSTALL_NAME_DIR ${LIBS_DIR})



# Win32 delivered packages
if(WIN32)
  set(MYSQL_INCLUDE_DIR ${CMAKE_SOURCE_DIR}/dep/include/mysql)
  set(MYSQL_LIBRARY ${CMAKE_SOURCE_DIR}/dep/lib/${DEP_ARCH}_release/libmySQL.lib)
  set(MYSQL_DEBUG_LIBRARY ${CMAKE_SOURCE_DIR}/dep/lib/${DEP_ARCH}_debug/libmySQL.lib)
  set(OPENSSL_INCLUDE_DIR ${CMAKE_SOURCE_DIR}/dep/include/openssl)
  set(OPENSSL_LIBRARIES ${CMAKE_SOURCE_DIR}/dep/lib/${DEP_ARCH}_release/libeay32.lib)
  set(OPENSSL_DEBUG_LIBRARIES ${CMAKE_SOURCE_DIR}/dep/lib/${DEP_ARCH}_debug/libeay32.lib)
  set(BOOST_REQUIRED_VERSION 1.73)
  # zlib is build
endif()

# *nix-specific packages
if(UNIX)
  set(BOOST_REQUIRED_VERSION 1.71)
  if(POSTGRESQL)
    find_package(PostgreSQL REQUIRED)

    if(POSTGRESQL_FOUND)
      include_directories(${POSTGRESQL_INCLUDE_DIRS})
    endif(POSTGRESQL_FOUND)

  else()
    find_package(MySQL REQUIRED)
  endif()

  find_package(OpenSSL 1.0 REQUIRED)
  find_package(ZLIB REQUIRED)
endif()

# Find needed packages and if necessery abort if something important is missing
find_package(Boost ${BOOST_REQUIRED_VERSION} REQUIRED COMPONENTS system filesystem program_options thread iostreams regex)

if(Boost_FOUND)
  include_directories(${Boost_INCLUDE_DIRS})
else()
  message(FATAL_ERROR "This project requires boost.  Please install from http://www.boost.org")
endif()

# Find core revision
if(GIT_EXECUTABLE)
  execute_process(
    COMMAND ${GIT_EXECUTABLE} rev-parse HEAD
    WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
    OUTPUT_VARIABLE GIT_REVISION
    RESULT_VARIABLE GIT_RESULT
    ERROR_QUIET
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  if(GIT_RESULT)
    set(GIT_REVISION "Git repository not found")
  endif()
else()
  set(GIT_REVISION "Git not found")
endif()

if(DEFINED INCLUDE_BINDINGS_DIR AND INCLUDE_BINDINGS_DIR)
  # check if the directory exists
  if(NOT IS_DIRECTORY ${CMAKE_SOURCE_DIR}/src/bindings/${INCLUDE_BINDINGS_DIR})
    message(FATAL_ERROR "Could not find the script library which was supposed to be: " ${CMAKE_SOURCE_DIR}/src/bindings/${INCLUDE_BINDINGS_DIR})
  endif()
  # check if it really contains a CMakeLists.txt
  if(NOT EXISTS ${CMAKE_SOURCE_DIR}/src/bindings/${INCLUDE_BINDINGS_DIR}/CMakeLists.txt)
    message(FATAL_ERROR "The script library does not contain a CMakeLists.txt!")
  endif()
else()
endif()

#if(PCH AND NOT PCHSupport_FOUND)
#  set(PCH 0 CACHE BOOL
#    "Use precompiled headers"
#    FORCE)
#  message(WARNING "No PCH for your system possible but PCH was set to 1. Resetting it."
#  )
#endif()

set(DEFINITIONS_RELEASE NDEBUG)
set(DEFINITIONS_DEBUG _DEBUG MANGOS_DEBUG)

if(DEBUG)
  set(CMAKE_BUILD_TYPE Debug)
else()
  set(CMAKE_BUILD_TYPE Release)
endif()

# print out the results before continuing
include(cmake/showoptions.cmake)

if(NOT BUILD_CORE AND NOT BUILD_EXTRACTOR AND NOT BUILD_VMAP_EXTRACTOR AND NOT BUILD_MMAP_EXTRACTOR)
  message(FATAL_ERROR "You must select something to build!")
endif()

# Generate revision-extractor
set(GENREV_SRC
  src/tools/genrevision/genrevision.cpp
)

add_executable(genrev
  ${GENREV_SRC}
)

add_custom_target("revision.h" ALL
  COMMAND genrev ${CMAKE_SOURCE_DIR}
  WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
  DEPENDS genrev
)

if(WIN32)
  install(
    FILES
      ${CMAKE_SOURCE_DIR}/dep/lib/${DEP_ARCH}_release/libeay32.dll
      ${CMAKE_SOURCE_DIR}/dep/lib/${DEP_ARCH}_release/libmySQL.dll
    DESTINATION ${LIBS_DIR}
    CONFIGURATIONS Release
  )
  install(
    FILES
      ${CMAKE_SOURCE_DIR}/dep/lib/${DEP_ARCH}_debug/libeay32.dll
      ${CMAKE_SOURCE_DIR}/dep/lib/${DEP_ARCH}_debug/libmySQL.dll
    DESTINATION ${LIBS_DIR}
    CONFIGURATIONS Debug
  )
  if(PLATFORM MATCHES X86)
    # Copy dll's Windows needs
    install(
      FILES
        ${CMAKE_SOURCE_DIR}/dep/lib/win32_release/dbghelp.dll
      DESTINATION ${LIBS_DIR}
      CONFIGURATIONS Release
    )
    install(
      FILES
        ${CMAKE_SOURCE_DIR}/dep/lib/win32_debug/dbghelp.dll
      DESTINATION ${LIBS_DIR}
      CONFIGURATIONS Debug
    )
  endif()
endif()

if(XCODE)
  if(PLATFORM MATCHES X86)
    set(CMAKE_OSX_ARCHITECTURES i386)
  else()
    set(CMAKE_OSX_ARCHITECTURES x86_64)
  endif()
endif()

list(APPEND CMAKE_MODULE_PATH
        ${CMAKE_SOURCE_DIR}/cmake/macros)

add_subdirectory(dep)

# Add definitions for all build types
# Don't place this above 'dep' subdirectory! Because of defines build will crash.
set(DEFINITIONS
  SYSCONFDIR="${CMAKE_INSTALL_PREFIX}/${CONF_FOLDER_NAME}/"
)

if(POSTGRESQL)
  set(DEFINITIONS ${DEFINITIONS} DO_POSTGRESQL)
else()
  set(DEFINITIONS ${DEFINITIONS} DO_MYSQL)
endif()

if(WIN32)
  set(DEFINITIONS ${DEFINITIONS} WIN32 _WIN32)
endif()

if(DEBUG)
  set_directory_properties(PROPERTIES COMPILE_DEFINITIONS "${DEFINITIONS};${DEFINITIONS_DEBUG}")
else()
  set_directory_properties(PROPERTIES COMPILE_DEFINITIONS "${DEFINITIONS};${DEFINITIONS_RELEASE}")
endif()

if(BUILD_EXTRACTOR)
  add_subdirectory(contrib/extractor)
endif()

if(BUILD_VMAP_EXTRACTOR)
  add_subdirectory(contrib/vmap_extractor)
  add_subdirectory(contrib/vmap_assembler)
endif()

if(BUILD_MMAP_EXTRACTOR)
  add_subdirectory(contrib/mmap)
endif()
# if(SQL)
#   add_subdirectory(sql)
# endif()

add_subdirectory(app)
add_subdirectory(src)