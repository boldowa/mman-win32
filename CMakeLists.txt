
cmake_minimum_required(VERSION 3.6)
project(mman C)

set(VERSION "0.5.0")

option(MMAN_TEST   "Executes test [default: OFF]" OFF)
option(MMAN_SHARED "Generate shared object [default: OFF]" OFF)
option(MMAN_COVERAGE "Check coverage [default: OFF]" OFF)
option(MMAN_STATIC "Force static build [default: OFF]" OFF)

if(NOT "Windows" STREQUAL ${CMAKE_SYSTEM_NAME})
	message(FATAL_ERROR "Windows only...")
endif(NOT "Windows" STREQUAL ${CMAKE_SYSTEM_NAME})

macro(set_cflags tgt)
	set(extra_argv ${ARGN})
	list(LENGTH extra_argv extra_argc)
	if(extra_argc GREATER 0)
		foreach(arg IN LISTS extra_argv)
			set(flg "${flg} ${arg}")
		endforeach()
	endif(extra_argc GREATER 0)
	set_target_properties(${tgt} PROPERTIES COMPILE_FLAGS "${flg}")
	unset(flg)
endmacro(set_cflags tgt flags)

# Set compile flags
set(MMAN_FLAGS "")
if(NOT MSVC)
	set(MMAN_FLAGS "${MMAN_FLAGS} -Wall -Wpedantic")
	set(CMAKE_EXE_LINKER_FLAGS_RELEASE "-s -Wl,--gc-sections")
	set(CMAKE_SHARED_LINKER_FLAGS_RELEASE "-s -Wl,--gc-sections")
	if(MMAN_STATIC)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -static")
	endif(MMAN_STATIC)
endif(NOT MSVC)
if(NOT CMAKE_BUILD_TYPE)
	message(STATUS "No build type selected, default to Release")
	set(CMAKE_BUILD_TYPE "Release" CACHE STRING 
                "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif(NOT CMAKE_BUILD_TYPE)

find_program(GCOV gcov)
if(NOT GCOV OR NOT MMAN_TEST OR NOT "Debug" STREQUAL CMAKE_BUILD_TYPE)
	message("MMAN_COVERAGE is disabled...")
	set(MMAN_COVERAGE OFF)
endif(NOT GCOV OR NOT MMAN_TEST OR NOT "Debug" STREQUAL CMAKE_BUILD_TYPE)


set(MMAN_HEADERS
	mman.h)
set(MMAN_PC
	${CMAKE_CURRENT_BINARY_DIR}/mman.pc)

set(INSTALL_BIN_DIR "${CMAKE_INSTALL_PREFIX}/bin" CACHE PATH "Installation directory for executables")
set(INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib" CACHE PATH "Installation directory for libraries")
set(INSTALL_INC_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE PATH "Installation directory for headers")
set(INSTALL_MAN_DIR "${CMAKE_INSTALL_PREFIX}/share/man" CACHE PATH "Installation directory for manual pages")
set(INSTALL_PKGCONFIG_DIR "${CMAKE_INSTALL_PREFIX}/share/pkgconfig" CACHE PATH "Installation directory for pkgconfig (.pc) files")

configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/mman.pc.cmakein
		${MMAN_PC} @ONLY)

if(MMAN_SHARED)
	set(LIBMODE SHARED)
endif(MMAN_SHARED)
add_library(mman ${LIBMODE} mman.c)
set_cflags(mman ${MMAN_FLAGS})
if(MMAN_COVERAGE)
	set_cflags(mman ${MMAN_FLAGS} "--coverage")
	target_link_libraries(mman gcov)
endif(MMAN_COVERAGE)

if(MMAN_TEST)
	# Search wine
	if(NOT "Windows" STREQUAL ${CMAKE_HOST_SYSTEM_NAME})
		find_program(WINE wine)
		if(NOT WINE)
			message(FATAL_ERROR "wine isn't installed...")
		endif(NOT WINE)
	endif(NOT "Windows" STREQUAL ${CMAKE_HOST_SYSTEM_NAME})

	# make
	add_executable(mman_test test.c)
	target_link_libraries(mman_test mman)
	set_cflags(mman_test ${MMAN_FLAGS})

	# Test
	enable_testing()
	add_test(NAME MMAN_Test COMMAND $<TARGET_FILE:mman_test>)

	# check
	add_custom_target(check ALL
		COMMAND ${CMAKE_COMMAND} -E echo "-- starts test..."
		COMMAND ${WINE} $<TARGET_FILE:mman_test>
		COMMAND ${CMAKE_COMMAND} -E echo "-- done."
		)
	add_dependencies(check mman_test)
	# Coverage check
	if(MMAN_COVERAGE)
		add_custom_target(coverage ALL
			COMMAND ${CMAKE_COMMAND} -E echo "-- execute gcov..."
			COMMAND find CMakeFiles/ -type f -name "*.gcda" -print0 | xargs -0 env LANG=C gcov
			COMMAND ${CMAKE_COMMAND} -E echo "-- done."
			)
		add_dependencies(coverage check)
	endif(MMAN_COVERAGE)
endif(MMAN_TEST)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL )
    install(TARGETS mman
        RUNTIME DESTINATION "${INSTALL_BIN_DIR}"
        ARCHIVE DESTINATION "${INSTALL_LIB_DIR}"
        LIBRARY DESTINATION "${INSTALL_LIB_DIR}" )
endif()
if(NOT SKIP_INSTALL_HEADERS AND NOT SKIP_INSTALL_ALL )
	install(FILES ${MMAN_HEADERS} DESTINATION "${INSTALL_INC_DIR}/sys")
endif()
if(NOT SKIP_INSTALL_FILES AND NOT SKIP_INSTALL_ALL )
	install(FILES ${MMAN_PC} DESTINATION "${INSTALL_PKGCONFIG_DIR}")
endif()

