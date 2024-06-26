# ~~~
# Summary:      Main plugin build script
# Copyright (c) 2020-2021 Mike Rossiter
# License:      GPLv3+
# ~~~

# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 3 of the License, or
# (at your option) any later version.

# -------- Cmake setup ---------
#
cmake_minimum_required(VERSION 3.12.0)
include(${CMAKE_SOURCE_DIR}/cmake/CmakeSetup.cmake NO_POLICY_SCOPE)
message(STATUS "Cmake version: ${CMAKE_VERSION}.")


# -------- Build setup  --------
#
include(PluginOptions)
if (BUILD_TYPE)
  message(STATUS "Building: ${BUILD_TYPE}")
else ()
  message(STATUS "Configuring")
endif ()

# -------- Plugin setup ----------
#
include(Plugin.cmake)

# -------- Setup completed, build the plugin --------
#
#if ("${OCPN_TARGET_TUPLE}" MATCHES wx32)
  # Evil, dirty hack to force a different name for wx32 builds. We
  # need a cleaner solution here. FIXME(leamas)
#  set(PKG_NAME "${PKG_NAME}-wx32")
#endif ()

project(${PKG_NAME} VERSION ${PKG_VERSION})
include(PluginCompiler)

add_library(${CMAKE_PROJECT_NAME} SHARED EXCLUDE_FROM_ALL ${SRC})
include_directories(BEFORE ${CMAKE_BINARY_DIR}/include)

add_subdirectory("opencpn-libs/${PKG_API_LIB}")
if (${PKG_NAME} MATCHES wx32)
  target_link_libraries(${CMAKE_PROJECT_NAME} ocpn::api_wx32)
else ()
  target_link_libraries(${CMAKE_PROJECT_NAME} ocpn::api)
endif ()

set(VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
set(VERSION_MINOR ${PROJECT_VERSION_MINOR})
set(VERSION_PATCH ${PROJECT_VERSION_PATCH})

set(PACKAGE_NAME ${CMAKE_PROJECT_NAME})

# Create the minimim ocpn version id string like ov50
string(REGEX REPLACE "\([0-9]\)\.\([0-9]\).*" "ov\\1\\2"
       OCPN_MIN_VERSION ${API_OCPN_MIN_VERSION})

if (COMMAND late_init)
  late_init()
endif ()

# Configure the flatpak manifest
file(
  GLOB manifest ${PROJECT_SOURCE_DIR}/flatpak/org.opencpn.OpenCPN.Plugin.*.yaml
)
include(ConfigureManifest)
configure_manifest(${manifest} _new_manifest)

# Set up targets. Targets sets up new targets with BUILD_TYPE set to
# 'flatpak', 'android' or 'tarball'. The initial call without BUILD_TYPE ends
# here.
#
include(Targets)
create_targets(${_new_manifest})

if ("${BUILD_TYPE}" STREQUAL "")
  message(STATUS "****EXIT 1")
  return ()
endif ()

message(STATUS "****PASS 2")

if (NOT ${BUILD_TYPE} STREQUAL "flatpak")
  # Build package as required (flatpak already dealt with).
  #
  include(PluginInstall)
  if (QT_ANDROID)
    include(AndroidLibs)
    #include_directories(${CMAKE_ANDROID_NDK}/sysroot/usr/include)
    #target_include_directories(${PACKAGE_NAME} PUBLIC "s/home/dsr/Android/Sdk/ndk-bundle/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/include")
  else ()
    include(PluginLibs)
  endif ()
  include(PluginLocalization)

  include_directories(BEFORE ${CMAKE_BINARY_DIR}/include)

  if (COMMAND add_plugin_libraries)
    add_plugin_libraries()
  endif ()

endif ()

configure_file(
  ${CMAKE_SOURCE_DIR}/config.h.in ${CMAKE_BINARY_DIR}/include//config.h
  @ONLY
)

configure_file(
  # The cloudsmith upload script
  ${CMAKE_SOURCE_DIR}/ci/upload.sh.in ${CMAKE_BINARY_DIR}/upload.sh
  @ONLY
)
configure_file(
  # The cloudsmith upload script, windows bat file.
  ${CMAKE_SOURCE_DIR}/ci/upload.bat.in ${CMAKE_BINARY_DIR}/upload.bat
  @ONLY
)

message(STATUS "At configure_file, pkg_target_arch: ${pkg_target_arch}.")

set(checksum "@checksum@")
configure_file(
  # The XML metadata file
  ${CMAKE_SOURCE_DIR}/plugin.xml.in
  ${CMAKE_BINARY_DIR}/${pkg_displayname}.xml.in
  @ONLY
)
