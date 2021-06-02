#==================================================================================
#	Copyright (c) 2021 AT&T Intellectual Property.
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.
#==================================================================================
#

# This CMake definition supports several -D command line options:
#
#	-DDEV_PKG=1			Development package configuration
#	-DBUILD_DOC=1		Man pages generated
#	-DGPROF=1			Enable profiling compile time flags
#	-D-DPRESERVE_PTYPE=1 to prevent changing the processor type to a standard string

project( libe2ap )
cmake_minimum_required( VERSION 3.5 )

#
#	Versioning is based on the E2 specification version number in play when the generated
#	code in src/e2ap was created.  The major/minor numbers should match the working group
#	name (e.g.  ORAN-WG3.E2AP-v01.00, would be 1.0). The patch version can be increased for
#	small tweaks and possibly corrections (omissions/inclusions etc.) as is needed.
#
set( major_version "1" )
set( minor_version "1" )
set( patch_level "0" )

set( install_root "${CMAKE_INSTALL_PREFIX}" )
set( install_inc "include/riclibe2ap" )
if( MAN_PREFIX )
	set( install_man ${MAN_PREFIX} )			# is there a cmake var for this -- can't find one
else()
	set( install_man "/usr/share/man" )			# this needs to be fixed so it's not hard coded
endif()

# Must use GNUInstallDirs to install libraries into correct
# locations on all platforms.
include( GNUInstallDirs )

# externals may install using LIBDIR as established by the gnu include; it varies from system
# to system, and we don't trust that it is always set, so we default to lib if it is missing.
#
if( NOT CMAKE_INSTALL_LIBDIR )
	set( CMAKE_INSTALL_LIBDIR "lib" )
endif()

set( install_lib "${CMAKE_INSTALL_LIBDIR}" )
message( "+++ riclibe2ap library install target directory: ${install_lib}" )

# ---------------- extract some things from git ------------------------------

# commit id for the version string
execute_process(
	COMMAND bash -c "git rev-parse --short HEAD|awk '{printf\"%s\", $0}'"
	OUTPUT_VARIABLE git_id
)

# version information for library names and version string
execute_process(
	COMMAND bash -c "git describe --tags --abbrev=0 HEAD 2>/dev/null | awk -v tag=0.0.4095 ' { tag=$1 } END{ print  tag suffix }'|sed 's/\\./;/g' "
	OUTPUT_VARIABLE mmp_version_str
	ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE
)
message( "+++ mmp version from tag: '${mmp_version_str}'" )

# extra indicator to show that the build was based on modified file(s) and not the true commit
# (no hope of reproducing the exact library for debugging). Used only for the internal version
# string.
execute_process(
	COMMAND bash -c "git diff --shortstat|awk -v fmt=%s -v r=-rotten '{ s=r } END { printf( fmt, s ) }'"
	OUTPUT_VARIABLE spoiled_str
)

# uncomment these lines once CI starts adding a tag on merge
#set( mmp_version ${mmp_version_str} )
#list( GET mmp_version 0 major_version )
#list( GET mmp_version 1 minor_version )
#list( GET mmp_version 2 patch_level )

# define constants used in the version string, debugging, etc.
add_definitions(
	-DGIT_ID=${git_id}
	-DMAJOR_VER=${major_version}
	-DMINOR_VER=${minor_version}
	-DPATCH_VER=${patch_level}
)

# ---------------- suss out pkg gen tools so we don't fail generating packages that the system cannot support --------------

# deb packages use underbars, and package manager(s) seem to flip the *_64 processor type
# to the old (non-standard) amd64 string, so we do it here for consistency. Set -DPRESERVE_PTYPE=1
# to prevent the flip. RPM packages will always be given the system generated processor type string.
#
if( ${CMAKE_SYSTEM_PROCESSOR} MATCHES "x86_64" )
	if( NOT PRESERVE_PTYPE )
		set( deb_sys_name "amd64" )
	else()
		set( deb_sys_name ${CMAKE_SYSTEM_PROCESSOR} )
	endif()
else()
	set( deb_sys_name ${CMAKE_SYSTEM_PROCESSOR} )
endif()
unset( PRESERVE_PTYPE CACHE )					# we don't want this to persist

set( rpm_sys_name ${CMAKE_SYSTEM_PROCESSOR} )

if( DEV_PKG )
	set( deb_pkg_name "riclibe2ap-dev" )
	set( rpm_pkg_name "riclibe2ap-devel" )
else()
	set( deb_pkg_name "riclibe2ap" )
	set( rpm_pkg_name "riclibe2ap" )
endif()

set( pkg_label "riclibe2ap${spoiled_str}-${major_version}.${minor_version}.${patch_level}-${sys_name}" )
set( rpm_pkg_label "${rpm_pkg_name}${spoiled_str}-${major_version}.${minor_version}.${patch_level}-${rpm_sys_name}" )
set( deb_pkg_label "${deb_pkg_name}${spoiled_str}_${major_version}.${minor_version}.${patch_level}_${deb_sys_name}" )
message( "+++ pkg name: ${deb_pkg_label}.deb" )

set( gen_rpm 0 )
find_program( rpm NAMES rpmbuild )					# rpm package gen requires this to be installed
if( "${rpm}" MATCHES "rpm-NOTFOUND" )				# cannot build rpm
	set( pkg_list "DEB" )
	message( "### make package will generate only deb package; cannot find support to generate rpm packages" )
else()
	message( "+++ pkg name: ${rpm_pkg_label}.rpm" )		# debugging if we think we can gen rpm too
	set( pkg_list "DEB;RPM" )
	set( gen_rpm 1 )
	message( "+++ make package will generate both deb and rpm packages" )
endif()



# bleeding cmake names are short novels; and when lines cannot be split they're a pain, so we create a synonym
set ( srcd "${CMAKE_CURRENT_SOURCE_DIR}" )

# this gets us round a chicken/egg problem. include files don't exist until make is run
# but Cmake insists on having these exist when we add them to include directories to
# enable code to find them after we build them.
#
# the source directories need to be explicitly added here
include_directories( "${srcd}/src/e2ap" "${srcd}/src/wrapper")

# Compiler flags
#
set( CMAKE_POSITION_INDEPENDENT_CODE ON )
set( CMAKE_C_FLAGS "-g -DASN_DISABLE_OER_SUPPORT" )
set( CMAKE_CPP_FLAGS "-g " )
set( CMAKE_CXX_FLAGS "-g " )
if( GPROF )					# if set, we'll set profiling flag on compiles
	message( "+++ profiling is on" )
	set( CMAKE_C_FLAGS "-pg " )
	set( CMAKE_CPP_FLAGS "-pg " )
else()
	message( "+++ profiling is off" )
endif()
unset( GPROF CACHE )					# we don't want this to persist


# --------------------- external building --------------------------------------------------------
# TESTING -- this likely can go, but until we are sure we don't have an external thing to pull/build
# keep it as an example
#execute_process( COMMAND  git submodule update --init -- ext/jsmn
#                WORKING_DIRECTORY  ${CMAKE_CURRENT_SOURCE_DIR}
#        )


# --------------------------------building --------------------------------------------------------
# Include modules
#
add_subdirectory( src/e2ap )
add_subdirectory( src/wrapper )

# shared and static libraries are built from the same object files.
#
add_library(  riclibe2ap SHARED
	"$<TARGET_OBJECTS:e2ap_objects>;$<TARGET_OBJECTS:wrapper_objects>;"
)
set_target_properties( riclibe2ap
	PROPERTIES
	OUTPUT_NAME "riclibe2ap"
	SOVERSION ${major_version}
	VERSION ${major_version}.${minor_version}.${patch_level}
)

# we only build/export the static archive (.a) if generating a dev package
if( DEV_PKG )
	add_library( riclibe2ap_static STATIC
		"$<TARGET_OBJECTS:e2ap_objects>;$<TARGET_OBJECTS:wrapper_objects>;"
	)
	set_target_properties( riclibe2ap_static
		PROPERTIES
		OUTPUT_NAME "riclibe2ap"
		SOVERSION ${major_version}
		VERSION ${major_version}.${minor_version}.${patch_level}
	)
endif()

# -------- unit testing -------------------------------------------------------
#	There are no real tests as the code is generated, but CI might need/want to
#	be able to run 'make test'.  This allows for it.
#
enable_testing()
add_test(
	NAME drive_unit_tests
	COMMAND bash ../test/unit_test.sh -q CMBUILD=${CMAKE_CURRENT_BINARY_DIR}
	WORKING_DIRECTORY ../test
)


# ------------- packaging -----------------------------------------------------

# Define what should be installed, and where they should go. For dev package we install
# only the RMr headers, man pages and archive (.a) files.  The run-time package gets just
# the library (.so) files and nothing more.
#
if( DEV_PKG )
	set( target_list "riclibe2ap_static" )
else()
	set( target_list "riclibe2ap" )
endif()

install( TARGETS ${target_list} EXPORT LibraryConfig
	LIBRARY  DESTINATION ${install_lib}
	ARCHIVE  DESTINATION ${install_lib}
	PUBLIC_HEADER DESTINATION ${install_inc}
)

unset( DEV_PKG  CACHE )			# prevent from being a hidden setting if user redoes things

IF( EXISTS "${CMAKE_ROOT}/Modules/CPack.cmake" )
	include( InstallRequiredSystemLibraries )

	set( CPACK_DEBIAN_PACKAGE_NAME ${deb_pkg_name} )
	set( CPACK_RPM_PACKAGE_NAME ${rpm_pkg_name} )

	set( CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION "/usr/local;/usr/local/bin;/usr/local/include;/usr/local/share;/usr/local/lib" )

	set( CPACK_set_DESTDIR "on" )
	set( CPACK_PACKAGING_INSTALL_PREFIX "${install_root}" )
	set( CPACK_GENERATOR "${pkg_list}" )

	set( CPACK_PACKAGE_DESCRIPTION "C++ framework for RIC xAPPs based on RMR." )
	set( CPACK_PACKAGE_DESCRIPTION_SUMMARY "RIC xAPP C++ framework library" )
	set( CPACK_PACKAGE_VENDOR "None" )
	set( CPACK_PACKAGE_CONTACT "None" )
	set( CPACK_PACKAGE_VERSION_MAJOR "${major_version}" )
	set( CPACK_PACKAGE_VERSION_MINOR "${minor_version}" )
	set( CPACK_PACKAGE_VERSION_PATCH "${patch_level}" )
	set( CPACK_PACKAGE "${pkg_label}" )						# generic name for old versions of cpack
	set( CPACK_DEBIAN_FILE_NAME "${deb_pkg_label}.deb" )
	set( CPACK_RPM_FILE_NAME "${rpm_pkg_label}.rpm" )

	# Future: define dependencies on RMR and other libs

	set( CPACK_DEBIAN_PACKAGE_PRIORITY "optional" )
	set( CPACK_DEBIAN_PACKAGE_SECTION "ric" )
	set( CPACK_DEBIAN_ARCHITECTURE ${CMAKE_SYSTEM_PROCESSOR} )
	set( CPACK_RPM_ARCHITECTURE ${CMAKE_SYSTEM_PROCESSOR} )

	# this seems ingnored if included
	#set( CPACK_COMPONENTS_ALL Libraries ApplicationData )

	INCLUDE( CPack )
ENDIF()
