############################################################################
# CMakeLists.txt
# Copyright (C) 2017  Belledonne Communications, Grenoble France
#
############################################################################
#
# This program is free software; you can redistribute it and/or
# modify it under the terms of the GNU General Public License
# as published by the Free Software Foundation; either version 2
# of the License, or (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA  02110-1301, USA.
#
############################################################################

add_definitions(-D_CRT_SECURE_NO_WARNINGS)

find_package(BelleSIP REQUIRED CONFIG)

add_definitions(-D_CRT_SECURE_NO_WARNINGS)

if(ENABLE_SHARED)
	set(LIME_LIBRARIES_FOR_TESTER lime)
else()
	set(LIME_LIBRARIES_FOR_TESTER lime-static)
endif()
set(HEADER_FILES_CXX lime-tester.hpp lime-tester-utils.hpp)
set(SOURCE_FILES_CXX
	lime-tester.cpp
	lime-tester-utils.cpp
	lime_double_ratchet-tester.cpp
	lime_lime-tester.cpp
	lime_helloworld-tester.cpp
	lime_crypto-tester.cpp
	lime_massive_group-tester.cpp
)

set(SOURCE_FILES_C
	lime_ffi-tester.c
)

bc_apply_compile_flags(SOURCE_FILES_C STRICT_OPTIONS_CPP STRICT_OPTIONS_C)
bc_apply_compile_flags(SOURCE_FILES_CXX STRICT_OPTIONS_CPP STRICT_OPTIONS_CXX)

set (THREADS_PREFER_PTHREAD_FLAG ON)
find_package (Threads REQUIRED)

if(ANDROID OR IOS)
	add_library(limetester SHARED ${HEADER_FILES_CXX} ${SOURCE_FILES_CXX} ${SOURCE_FILES_C} )
	target_link_libraries(limetester PRIVATE bctoolbox bctoolbox-tester ${BELLESIP_TARGETNAME} ${LIME_LIBRARIES_FOR_TESTER} ${SOCI_LIBRARIES} ${SOCI_sqlite3_PLUGIN} ${CMAKE_THREAD_LIBS_INIT})
	if(IOS)
		target_link_libraries(limetester PRIVATE sqlite3)
		set(MIN_OS ${LINPHONE_IOS_DEPLOYMENT_TARGET})
		set_target_properties(limetester PROPERTIES
			FRAMEWORK TRUE
			MACOSX_FRAMEWORK_IDENTIFIER com.belledonne-communications.limetester
			MACOSX_FRAMEWORK_INFO_PLIST "${CMAKE_SOURCE_DIR}/build/osx/Info.plist.in"
			PUBLIC_HEADER "${HEADER_FILES_CXX}"
		)
	endif()
	install(TARGETS limetester
		RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
		LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
		ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
		FRAMEWORK DESTINATION Frameworks
		PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
		)
	install(FILES ${HEADER_FILES_CXX}
		DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/lime
		PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
	)
else()
	add_executable(lime_tester ${SOURCE_FILES_CXX} ${HEADER_FILES_CXX} ${SOURCE_FILES_C} )
	set_target_properties(lime_tester PROPERTIES LINKER_LANGUAGE CXX)
	target_link_libraries(lime_tester PRIVATE bctoolbox bctoolbox-tester ${BELLESIP_TARGETNAME} ${LIME_LIBRARIES_FOR_TESTER} ${SOCI_LIBRARIES} ${SOCI_sqlite3_PLUGIN} ${CMAKE_THREAD_LIBS_INIT})

	# Some tests suite need a local X3DH server running in default config, so run them suite by suite to at least be able to pass some if no server can be found
	add_test(NAME crypto COMMAND lime_tester --verbose --resource-dir ${CMAKE_CURRENT_SOURCE_DIR} --suite "Crypto")
	add_test(NAME double_ratchet COMMAND lime_tester --verbose --resource-dir ${CMAKE_CURRENT_SOURCE_DIR} --suite "double ratchet")
	add_test(NAME hello_world COMMAND lime_tester --verbose --resource-dir ${CMAKE_CURRENT_SOURCE_DIR} --suite "Hello World")
	add_test(NAME lime COMMAND lime_tester --verbose --resource-dir ${CMAKE_CURRENT_SOURCE_DIR} --suite "lime")
	if (ENABLE_C_INTERFACE)
		add_test(NAME C_ffi COMMAND lime_tester --verbose --resource-dir ${CMAKE_CURRENT_SOURCE_DIR} --suite "FFI")
	endif()

	if(ENABLE_PROFILING)
		set_target_properties(lime_tester PROPERTIES LINK_FLAGS "-pg")
	endif()
endif()

if (ENABLE_JNI)
	add_subdirectory(java)
endif()
