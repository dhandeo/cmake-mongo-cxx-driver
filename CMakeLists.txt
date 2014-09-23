cmake_minimum_required(VERSION 2.8)

# Basic build options for building the driver.
set (BUILD_SHARED_LIBS ON)
option(SSL "SSL" ON)


# We need to know where boost is, both to inform the build of mongo-cxx-driver, and
# so we can use it in our tests, so find it, and add it to our include and lib paths.
find_package (Boost
  REQUIRED
  COMPONENTS filesystem program_options system thread
)

# Include boost in projects
include_directories (${Boost_INCLUDE_DIRS})
link_directories (${Boost_LIBRARY_DIRS})

include (ExternalProject)

# Setup default flags
set (MONGOCLIENT_SCONS_FLAGS
  ${MONGOCLIENT_SCONS_FLAGS}
  --warn=no-missing-sconscript
  --cache
  --use-system-boost
  --cpppath=${Boost_INCLUDE_DIRS}
  --libpath=${Boost_LIBRARY_DIRS}
  --warn=no-all
)

# On Mavericks, presume that boost is linked to libc++
if (APPLE)
  if(NOT ${CMAKE_SYSTEM_VERSION} VERSION_LESS "13")
    list (APPEND MONGOCLIENT_SCONS_FLAGS --osx-version-min=10.8 --libc++)
  endif ()
endif ()

if (NOT WIN32)
    list (APPEND MONGOCLIENT_SCONS_FLAGS
        --cc=${CMAKE_C_COMPILER}
        --cxx=${CMAKE_CXX_COMPILER}
    )
endif ()

# If we are doing a debug build, then build mongoclient the same way
if (CMAKE_BUILD_TYPE MATCHES Debug)
    list (APPEND MONGOCLIENT_SCONS_FLAGS --dbg=on)
endif ()

if (BUILD_SHARED_LIBS)
  list (APPEND MONGOCLIENT_SCONS_FLAGS --sharedclient)
endif ()

# If we are building with SSL, enable it for scons
if (SSL)
    list (APPEND MONGOCLIENT_SCONS_FLAGS
        --ssl
        --use-sasl-client
    )
endif ()


find_program(SCONS_BINARY
    NAME scons scons.bat
    PATHS C:/Python27/Scripts /usr/bin/scons
)

# Pull the project and build
ExternalProject_Add(
  mongo-cxx-driver
  GIT_REPOSITORY git://github.com/mongodb/mongo-cxx-driver.git
  GIT_TAG 26compat
  LOG_DOWNLOAD ON
  LOG_INSTALL ON
  LOG_BUILD ON
  CONFIGURE_COMMAND ""
  BUILD_IN_SOURCE 1
  BUILD_COMMAND ${SCONS_BINARY} ${MONGOCLIENT_SCONS_FLAGS} mongoclient
  INSTALL_COMMAND ""
)

# NOTE: The following should not be necessary if libmongoclient had a properly constructed
# config.h header, see CXX-23.
add_definitions (-DMONGO_SSL)

if (CMAKE_BUILD_TYPE MATCHES Debug)
    add_definitions(-D_DEBUG)
endif ()

# Add googletest include dir
ExternalProject_Get_Property (mongo-cxx-driver source_dir)
include_directories (${source_dir}/include)

# Add googletest library dir
ExternalProject_Get_Property (mongo-cxx-driver binary_dir)
link_directories (${binary_dir})
