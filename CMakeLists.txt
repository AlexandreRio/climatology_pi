##---------------------------------------------------------------------------
## Author:      Sean D'Epagnier
##***************************************************************************
## *   Copyright (C) 2013 by Sean D'Epagnierer                               *
## *   This program is free software; you can redistribute it and/or modify  *
## *   it under the terms of the GNU General Public License as published by  *
## *   the Free Software Foundation; either version 3 of the License, or     *
## *   (at your option) any later version.                                   *
## *                                                                         *
## *   This program is distributed in the hope that it will be useful,       *
## *   but WITHOUT ANY WARRANTY; without even the implied warranty of        *
## *   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the         *
## *   GNU General Public License for more details.                          *
## *                                                                         *
## *   You should have received a copy of the GNU General Public License     *
## *   along with this program; if not, write to the                         *
## *   Free Software Foundation, Inc.,                                       *
## *   51 Franklin Street, Fifth Floor, Boston, MA 02110-1301,  USA.             *
## ***************************************************************************

# define minimum cmake version
CMAKE_MINIMUM_REQUIRED(VERSION 2.6.2)
# This should be 2.8.0 to have FindGTK2 module
IF (COMMAND cmake_policy)
  CMAKE_POLICY(SET CMP0003 OLD)
  CMAKE_POLICY(SET CMP0005 OLD)
  CMAKE_POLICY(SET CMP0011 OLD)
ENDIF (COMMAND cmake_policy)

PROJECT(climatology_pi)

SET(PACKAGE_NAME climatology_pi)
SET(PLUGIN_SOURCE_DIR .)
MESSAGE (STATUS "")
MESSAGE (STATUS "*** Staging to build ${PACKAGE_NAME} ***")

#SET(CMAKE_BUILD_TYPE Debug)

IF(UNIX AND NOT APPLE)
  SET(PREFIX_PLUGINS /usr/lib/opencpn)
  SET(PREFIX_DATA /usr/share)
ENDIF(UNIX AND NOT APPLE)

INCLUDE_DIRECTORIES(${CMAKE_SOURCE_DIR}/include ${CMAKE_SOURCE_DIR}/src ${CMAKE_SOURCE_DIR}/wxJSON)

# require proper c++
#ADD_DEFINITIONS( "-Wall -ansi -pedantic -Wno-variadic-macros" )
#TODO: Should we use  -fno-stack-protector
#  IF NOT DEBUGGING CFLAGS="-O2 -march=native"
IF(NOT WIN32)
 ADD_DEFINITIONS( "-Wall -O2 -fexceptions -fvisibility=hidden" )

 IF(NOT APPLE)
  SET(CMAKE_SHARED_LINKER_FLAGS "-Wl,-Bsymbolic")
 ELSE(NOT APPLE)
  SET(CMAKE_SHARED_LINKER_FLAGS "-Wl -undefined dynamic_lookup")
 ENDIF(NOT APPLE)

ENDIF(NOT WIN32)

# Add some definitions to satisfy MS
IF(WIN32)
    ADD_DEFINITIONS(-D__MSVC__)
    ADD_DEFINITIONS(-D_CRT_NONSTDC_NO_DEPRECATE -D_CRT_SECURE_NO_DEPRECATE)
ENDIF(WIN32)

IF(NOT DEFINED wxWidgets_USE_FILE)
  SET(wxWidgets_USE_LIBS base core net xml html adv)
  SET(BUILD_SHARED_LIBS TRUE)
  FIND_PACKAGE(wxWidgets REQUIRED)
ENDIF(NOT DEFINED wxWidgets_USE_FILE)

FIND_PACKAGE(wxWidgets REQUIRED)
  INCLUDE(${wxWidgets_USE_FILE})
FIND_PACKAGE(Gettext REQUIRED)

# For convenience we define the sources as a variable. You can add
# header files and cpp/c files and CMake will sort them out


SET(SRC_CLIMATOLOGY
            src/ClimatologyOverlayFactory.cpp
            src/ClimatologyOverlayFactory.h
            src/climatology_pi.cpp
            src/climatology_pi.h
            src/ClimatologyUI.cpp
            src/ClimatologyUI.h
            src/ClimatologyDialog.cpp
            src/ClimatologyDialog.h
            src/ClimatologyConfigDialog.cpp
            src/ClimatologyConfigDialog.h
            src/zuFile.cpp
            src/zuFile.h
            src/IsoBarMap.cpp
            src/IsoBarMap.h
            src/icons.cpp
            src/icons.h

            wxJSON/jsonval.cpp
            wxJSON/jsonreader.cpp
            wxJSON/jsonwriter.cpp
)
ADD_LIBRARY(${PACKAGE_NAME} SHARED ${SRC_CLIMATOLOGY})

IF(NOT UNIX)
    SET(SRC_BZIP_CLIM
            src/bzip2/bzlib.c
            src/bzip2/blocksort.c
            src/bzip2/compress.c
            src/bzip2/crctable.c
            src/bzip2/decompress.c
            src/bzip2/huffman.c
            src/bzip2/randtable.c
    )
    ADD_LIBRARY(LIB_BZIP_CLIM ${SRC_BZIP_CLIM})

    INCLUDE_DIRECTORIES(${PLUGIN_SOURCE_DIR}/src/zlib-1.2.3)
    INCLUDE_DIRECTORIES(${PLUGIN_SOURCE_DIR}/src/bzip2)
ENDIF(NOT UNIX)

IF(WIN32)
  SET(OPENCPN_IMPORT_LIB "../../${CMAKE_CFG_INTDIR}/${PARENT}")
  TARGET_LINK_LIBRARIES( ${PACKAGE_NAME} LIB_BZIP_CLIM ${OPENCPN_IMPORT_LIB} )
  ADD_DEPENDENCIES(${PACKAGE_NAME} ${PARENT})
ENDIF(WIN32)

TARGET_LINK_LIBRARIES( ${PACKAGE_NAME} ${wxWidgets_LIBRARIES} )

IF(APPLE)
      INSTALL(TARGETS ${PACKAGE_NAME} BUNDLE LIBRARY DESTINATION ${CMAKE_BINARY_DIR}/OpenCPN.app/Contents/PlugIns)
ENDIF(APPLE)

IF(UNIX AND NOT APPLE)
  FIND_PACKAGE(BZip2 REQUIRED)
  INCLUDE_DIRECTORIES(${BZIP2_INCLUDE_DIR})
  FIND_PACKAGE(ZLIB REQUIRED)
  INCLUDE_DIRECTORIES(${ZLIB_INCLUDE_DIR})
  TARGET_LINK_LIBRARIES( ${PACKAGE_NAME} ${BZIP2_LIBRARIES} ${ZLIB_LIBRARY} )
  INSTALL(TARGETS ${PACKAGE_NAME} RUNTIME LIBRARY DESTINATION ${PREFIX_PLUGINS})
ENDIF(UNIX AND NOT APPLE)

IF(WIN32)
  INSTALL(TARGETS ${PACKAGE_NAME} RUNTIME DESTINATION "plugins")
ENDIF(WIN32)

IF(WIN32)
  INSTALL(DIRECTORY data DESTINATION "plugins\\\\climatology")
ELSE(WIN32)
  INSTALL(DIRECTORY data DESTINATION ${PREFIX_DATA}/plugins/climatology)
ENDIF(WIN32)

FUNCTION(PLUGIN_LOCALIZATION)
	FIND_PROGRAM(GETTEXT_XGETTEXT_EXECUTABLE xgettext)
	string(REPLACE "_pi" "" I18N_NAME ${PACKAGE_NAME})
	IF (GETTEXT_XGETTEXT_EXECUTABLE)
	ADD_CUSTOM_COMMAND(
	   OUTPUT po/${PACKAGE_NAME}.pot.dummy
	   COMMAND ${GETTEXT_XGETTEXT_EXECUTABLE} --force-po --package-name=${PACKAGE_NAME} --package-version="${PACKAGE_VERSION}" --output=po/${PACKAGE_NAME}.pot  --keyword=_ --width=80 --files-from=${CMAKE_CURRENT_SOURCE_DIR}/po/POTFILES.in
	   DEPENDS po/POTFILES.in po/${PACKAGE_NAME}.pot
	   WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
	   COMMENT "${I18N_NAME}-pot-update [${PACKAGE_NAME}]: Generated pot file."
	)
	ADD_CUSTOM_TARGET(${I18N_NAME}-pot-update COMMENT "[${PACKAGE_NAME}]-pot-update: Done." DEPENDS po/${PACKAGE_NAME}.pot.dummy)

	ENDIF(GETTEXT_XGETTEXT_EXECUTABLE )

	MACRO(GETTEXT_UPDATE_PO _potFile)
	   SET(_poFiles ${_potFile})
	   GET_FILENAME_COMPONENT(_absPotFile ${_potFile} ABSOLUTE)

	   FOREACH (_currentPoFile ${ARGN})
	      GET_FILENAME_COMPONENT(_absFile ${_currentPoFile} ABSOLUTE)
	      GET_FILENAME_COMPONENT(_poBasename ${_absFile} NAME_WE)

	      ADD_CUSTOM_COMMAND(
		 OUTPUT ${_absFile}.dummy
		 COMMAND ${GETTEXT_MSGMERGE_EXECUTABLE} --width=80 --strict --quiet --update --backup=none --no-location -s ${_absFile} ${_absPotFile}
		 DEPENDS ${_absPotFile} ${_absFile}
		 COMMENT "${I18N_NAME}-po-update [${_poBasename}]: Updated po file."
	      )

	      SET(_poFiles ${_poFiles} ${_absFile}.dummy)

	   ENDFOREACH (_currentPoFile )

	   ADD_CUSTOM_TARGET(${I18N_NAME}-po-update COMMENT "[${PACKAGE_NAME}]-po-update: Done." DEPENDS ${_poFiles})
	ENDMACRO(GETTEXT_UPDATE_PO)

	IF (GETTEXT_MSGMERGE_EXECUTABLE)
	FILE(GLOB PACKAGE_PO_FILES po/*.po)
	GETTEXT_UPDATE_PO(po/${PACKAGE_NAME}.pot ${PACKAGE_PO_FILES})
	ENDIF(GETTEXT_MSGMERGE_EXECUTABLE)

	SET(_gmoFiles)
	MACRO(GETTEXT_BUILD_MO)
	   FOREACH (_poFile ${ARGN})
	      GET_FILENAME_COMPONENT(_absFile ${_poFile} ABSOLUTE)
	      GET_FILENAME_COMPONENT(_poBasename ${_absFile} NAME_WE)
	      SET(_gmoFile ${CMAKE_CURRENT_BINARY_DIR}/${_poBasename}.mo)

	      ADD_CUSTOM_COMMAND(
		 OUTPUT ${_gmoFile}
		 COMMAND ${GETTEXT_MSGFMT_EXECUTABLE} --check -o ${_gmoFile} ${_absFile}
		 DEPENDS ${_absFile}
		 COMMENT "${I18N_NAME}-i18n [${_poBasename}]: Created mo file."
	      )

	IF(APPLE)
	      INSTALL(FILES ${_gmoFile} DESTINATION ${CMAKE_INSTALL_PREFIX}/bin/OpenCPN.app/Contents/Resources/${_poBasename}.lproj RENAME opencpn-${PACKAGE_NAME}.mo )
	ELSE(APPLE)
	      INSTALL(FILES ${_gmoFile} DESTINATION ${PREFIX_DATA}/locale/${_poBasename}/LC_MESSAGES RENAME opencpn-${PACKAGE_NAME}.mo )
	ENDIF(APPLE)

	      SET(_gmoFiles ${_gmoFiles} ${_gmoFile})
	   ENDFOREACH (_poFile )
	ENDMACRO(GETTEXT_BUILD_MO)

	if(GETTEXT_MSGFMT_EXECUTABLE)
	   FILE(GLOB PACKAGE_PO_FILES po/*.po)
	   GETTEXT_BUILD_MO(${PACKAGE_PO_FILES})
	   ADD_CUSTOM_TARGET(${I18N_NAME}-i18n COMMENT "${PACKAGE_NAME}-i18n: Done." DEPENDS ${_gmoFiles})
	   ADD_DEPENDENCIES(${PACKAGE_NAME} ${I18N_NAME}-i18n)
	ENDIF(GETTEXT_MSGFMT_EXECUTABLE)
ENDFUNCTION()
PLUGIN_LOCALIZATION()
