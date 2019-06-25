################################################################################
# Project:  lcms
# Purpose:  CMake build scripts
# Author:   Dmitry Baryshnikov, polimax@mail.ru
################################################################################
# Copyright (C) 2015-2019, NextGIS <info@nextgis.com>
# Copyright (C) 2012,2013,2014-2019 Dmitry Baryshnikov
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
# OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.
################################################################################
cmake_minimum_required (VERSION 2.8.12 FATAL_ERROR)

project(lcms)

if(NOT DEFINED PACKAGE_VENDOR)
    set(PACKAGE_VENDOR NextGIS)
endif()

if(NOT DEFINED PACKAGE_BUGREPORT)
    set(PACKAGE_BUGREPORT info@nextgis.com)
endif()


# some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

if(OSX_FRAMEWORK)
    set(INSTALL_BIN_DIR "bin" CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR "Library/Frameworks" CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(INSTALL_INC_DIR "${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Headers" CACHE INTERNAL "Installation directory for headers" FORCE)
    set(INSTALL_CMAKECONF_DIR ${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Resources/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
    set(SKIP_INSTALL_HEADERS ON)
    set(SKIP_INSTALL_EXECUTABLES ON)
    set(SKIP_INSTALL_FILES ON)
    set(SKIP_INSTALL_EXPORT ON)

    set(CMAKE_MACOSX_RPATH ON)
else()
    include(GNUInstallDirs)

    set(INSTALL_BIN_DIR ${CMAKE_INSTALL_BINDIR} CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR ${CMAKE_INSTALL_LIBDIR} CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(INSTALL_INC_DIR ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME} CACHE INTERNAL "Installation directory for headers" FORCE)
    set(INSTALL_CMAKECONF_DIR ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
endif()

# setup package meta-data
include(util)

check_version(LCMS_MAJOR_VERSION LCMS_MINOR_VERSION LCMS_REV_VERSION)
set(VERSION ${LCMS_MAJOR_VERSION}.${LCMS_MINOR_VERSION}.${LCMS_REV_VERSION})

report_version(${PROJECT_NAME} ${VERSION})

if(OSX_FRAMEWORK)
    set(FRAMEWORK_VERSION "${LCMS_MAJOR_VERSION}")
endif()

set(PACKAGE_NAME "lcms")
set(PACKAGE_STRING "${PACKAGE_NAME} ${VERSION}")
set(PACKAGE_TARNAME ${PROJECT_NANE})

if(CMAKE_GENERATOR_TOOLSET MATCHES "v([0-9]+)_xp")
    add_definitions(-D_WIN32_WINNT=0x0501)
endif()

find_package(Threads)
if(Threads_FOUND)
    add_definitions(-DHasTHREADS=1)
endif()

include (TestBigEndian)
test_big_endian(WORDS_BIGENDIAN)
if(WORDS_BIGENDIAN)
    add_definitions(-DWORDS_BIGENDIAN)
endif()

# setup paths
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src)

set(PUBLIC_HHEADERS include/lcms2.h include/lcms2_plugin.h)

set(CSOURCES
    src/cmscnvrt.c
    src/cmserr.c
    src/cmsgamma.c
    src/cmsgmt.c
    src/cmsintrp.c
    src/cmsio0.c
    src/cmsio1.c
    src/cmslut.c
    src/cmsplugin.c
    src/cmssm.c
    src/cmsmd5.c
    src/cmsmtrx.c
    src/cmspack.c
    src/cmspcs.c
    src/cmswtpnt.c
    src/cmsxform.c
    src/cmssamp.c
    src/cmsnamed.c
    src/cmscam02.c
    src/cmsvirt.c
    src/cmstypes.c
    src/cmscgats.c
    src/cmsps2.c
    src/cmsopt.c
    src/cmshalf.c
    src/cmsalpha.c
    )

set(HHEADERS ${PUBLIC_HHEADERS} src/lcms2_internal.h)

if(WIN32)
    set(HHEADERS ${HHEADERS} include/resource.h)
    set(CSOURCES ${CSOURCES} include/lcms2.rc ) # src/lcms2.def
endif()

set(LIB_NAME ${PROJECT_NAME})
if(BUILD_SHARED_LIBS)
    add_library(${LIB_NAME} SHARED ${CSOURCES} ${HHEADERS})
    if(MSVC)
        set_target_properties(${LIB_NAME} PROPERTIES IMPORT_SUFFIX "_i.lib")
    endif()

    set_target_properties (${LIB_NAME} PROPERTIES
        VERSION ${VERSION}
        SOVERSION ${LCMS_MAJOR_VERSION}
        DEFINE_SYMBOL CMS_DLL_BUILD)
elseif(OSX_FRAMEWORK)
    add_library(${LIB_NAME} SHARED ${CSOURCES} ${HHEADERS})
    set_target_properties(${LIB_NAME} PROPERTIES
        FRAMEWORK TRUE
        FRAMEWORK_VERSION "${FRAMEWORK_VERSION}"
        VERSION ${VERSION}
        MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${VERSION}
        MACOSX_FRAMEWORK_BUNDLE_VERSION ${VERSION}
        MACOSX_FRAMEWORK_IDENTIFIER org.jsonc.jsonc
        XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
        PUBLIC_HEADER "${PUBLIC_HHEADERS}")
else()
    add_library(${LIB_NAME} STATIC ${CSOURCES} ${HHEADERS})
endif()

string(TOUPPER ${PACKAGE_NAME} PACKAGE_UPPER_NAME)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    install(TARGETS ${LIB_NAME}
        EXPORT ${PACKAGE_UPPER_NAME}Targets
        RUNTIME DESTINATION ${INSTALL_BIN_DIR}
        LIBRARY DESTINATION ${INSTALL_LIB_DIR}
        ARCHIVE DESTINATION ${INSTALL_LIB_DIR}
        INCLUDES DESTINATION ${INSTALL_INC_DIR}
        FRAMEWORK DESTINATION ${INSTALL_LIB_DIR}
    )
endif()

# install headers
if(NOT SKIP_INSTALL_HEADERS AND NOT SKIP_INSTALL_ALL )
    install(FILES ${PUBLIC_HHEADERS} DESTINATION ${INSTALL_INC_DIR} COMPONENT headers)
endif()

# Testing ======================================================================
enable_testing()
if(BUILD_TESTING)
    file(GLOB TEST_FILES "testbed/*.ic?")
    file(COPY ${TEST_FILES} DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
    add_executable(testcms testbed/testcms2.c testbed/testplugin.c testbed/zoo_icc.c testbed/testcms2.h)
    target_link_libraries(testcms ${LIB_NAME})
    add_test(testcms testcms)

endif()

# uninstall
add_custom_target(uninstall COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

# Export package ===============================================================

# Add path to includes to build-tree export
target_include_directories(${LIB_NAME} INTERFACE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
)

# Add all targets to the build-tree export set
export(TARGETS ${LIB_NAME}
    FILE ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Targets.cmake)

if(REGISTER_PACKAGE)
    # Export the package for use from the build-tree
    # (this registers the build-tree with a global CMake-registry)
    export(PACKAGE ${PACKAGE_UPPER_NAME})
endif()

# Create the <Package>Config.cmake file
configure_file(cmake/PackageConfig.cmake.in
    ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake @ONLY)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    # Install the <Package>Config.cmake
    install(FILES
      ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake
      DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)

    # Install the export set for use with the install-tree
    install(EXPORT ${PACKAGE_UPPER_NAME}Targets DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)
endif()

# Archiving ====================================================================

set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

# Get cpack zip archive name
get_cpack_filename(${VERSION} PROJECT_CPACK_FILENAME)
set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include(CPack)
