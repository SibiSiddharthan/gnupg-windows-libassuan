#[[
   Copyright (c) 2021 Sibi Siddharthan

   Distributed under the MIT license.
   Refer to the LICENSE file at the root directory of the parent project
   for details.
]]

cmake_minimum_required(VERSION 3.12)
project(libassuan
		VERSION 2.5.4
		LANGUAGES C)

add_compile_definitions(HAVE_CONFIG_H)

set(PACKAGE \"${PROJECT_NAME}\")
set(PACKAGE_BUGREPORT \"https://bugs.gnupg.org\")
set(PACKAGE_NAME \"${PROJECT_NAME}\")
set(PACKAGE_STRING \"${PROJECT_NAME}\ ${PROJECT_VERSION}\")
set(PACKAGE_TARNAME \"${PROJECT_NAME}\")
set(PACKAGE_URL \"\")
set(PACKAGE_VERSION \"${PROJECT_VERSION}\")
set(VERSION \"${PROJECT_VERSION}\")
math(EXPR VERSION_HEX "(${PROJECT_VERSION_MAJOR} << 16) + (${PROJECT_VERSION_MINOR} << 8) + ${PROJECT_VERSION_PATCH}" OUTPUT_FORMAT HEXADECIMAL)

#find_package(GPGRT REQUIRED)
include(UtilityFunctions)

include(SignalHandler)
include(inline)

set(BUILD_REVISION \"\")
set(BUILD_TIMESTAMP \"\")

if(WIN32)
	set(HAVE_W32_SYSTEM 1)
	set(HAVE_W64_SYSTEM 1)
	set(HAVE_DOSISH_SYSTEM 1)
	set(HAVE_ISASCII 1)
	set(HAVE_PUTC_UNLOCKED 1)
endif()

check_c_headers(dlfcn.h fcntl.h inttypes.h locale.h memory.h stdint.h stdlib.h strings.h string.h
sys/select.h sys/socket.h sys/stat.h sys/time.h sys/types.h sys/ucred.h sys/uio.h ucred.h unistd.h
winsock2.h ws2tcpip.h)

check_symbol_exists(sys_siglist signal.h HAVE_DECL_SYS_SIGLIST)

if(WIN32)
	set(CMAKE_REQUIRED_LIBRARIES ws2_32)
endif()
check_functions(inet_pton flockfile fopencookie funlockfile funopen getaddrinfo
getpeereid getpeerucred getrlimit isascii memrchr nanosleep putc_unlocked setenv stat stpcpy)
unset(CMAKE_REQUIRED_LIBRARIES)

check_struct("struct cmsghdr" "cmsg_len" "stdlib.h;stddef.h;sys/socket.h;sys/types.h;sys/un.h" USE_DESCRIPTOR_PASSING)
check_struct("struct sockpeercred" "pid" "sys/socket.h;sys/types.h" HAVE_STRUCT_SOCKPEERCRED_PID)

if(WIN32)
	set(CMAKE_EXTRA_INCLUDE_FILES winsock2.h ws2tcpip.h)
else()
	set(CMAKE_EXTRA_INCLUDE_FILES sys/socket.h)
endif()
check_types(uintptr_t uint16_t socklen_t size_t)
unset(CMAKE_EXTRA_INCLUDE_FILES)

if(NOT HAVE_SIZE_T)
	set(size_t "unsigned\ int")
endif()

if(NOT HAVE_UINTPTR_T)
	set(uintptr_t "unsigned\ long\ long")
endif()

if(NOT HAVE_UINT16_T)
	set(uint16_t "unsigned\ short")
endif()

configure_file(libassuan-config.h.in config.h)

include_directories(${CMAKE_CURRENT_BINARY_DIR} src)

if(WIN32)
	set(host_os mingw32)
else()
	set(host_os linux-gnu)
endif()

if(WIN32)
	set(gen_headers_assuan_deps
		w32-includes.inc.h w32-types.inc.h w32-fd-t.inc.h w32-sock-nonce.inc.h w32-sys-pth-impl.h)
else()
	set(gen_headers_assuan_deps
	posix-includes.inc.h posix-types.inc.h posix-fd-t.inc.h posix-sock-nonce.inc.h posix-sys-pth-impl.h)
endif()
list(TRANSFORM gen_headers_assuan_deps PREPEND ${CMAKE_CURRENT_SOURCE_DIR}/src/)

add_executable(mkheader-assuan src/mkheader.c)
add_custom_command(OUTPUT ${CMAKE_BINARY_DIR}/assuan.h ${CMAKE_CURRENT_SOURCE_DIR}/src/assuan.h
		COMMAND mkheader-assuan ${host_os} ${CMAKE_CURRENT_SOURCE_DIR}/src/assuan.h.in ${PROJECT_VERSION} ${VERSION_HEX} > ${CMAKE_BINARY_DIR}/assuan.h
		COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/assuan.h ${CMAKE_CURRENT_SOURCE_DIR}/src/assuan.h
		DEPENDS mkheader-assuan ${CMAKE_CURRENT_SOURCE_DIR}/src/assuan.h.in ${gen_headers_assuan_deps})

add_custom_target(gen_headers_assuan DEPENDS ${CMAKE_BINARY_DIR}/assuan.h ${CMAKE_CURRENT_SOURCE_DIR}/src/assuan.h)

set(assuan_SOURCES
assuan-defs.h
assuan.c context.c system.c
debug.c debug.h conversion.c sysutils.c
client.c server.c
assuan-error.c
assuan-buffer.c
assuan-handler.c
assuan-inquire.c
assuan-listen.c
assuan-pipe-server.c
assuan-socket-server.c
assuan-pipe-connect.c
assuan-socket-connect.c
assuan-uds.c
assuan-logging.c
assuan-socket.c
assuan-io.c)

if(WIN32)
	list(APPEND assuan_SOURCES libassuan.def system-w32.c)
else()
	list(APPEND assuan_SOURCES system-posix.c funopen.c)
endif()

foreach(f isascii putc_unlocked memrchr stpcpy setenv)
	string(TOUPPER ${f} uf)
	string(PREPEND uf "HAVE_")
	if(NOT ${uf})
		list(APPEND assuan_SOURCES ${f}.c)
	endif()
endforeach()


list(TRANSFORM assuan_SOURCES PREPEND "src/")

if(WIN32)
	set(links ws2_32)
else()
	set(links pthread)
endif()
add_library(assuan SHARED ${assuan_SOURCES})
add_dependencies(assuan gen_headers_assuan)
target_link_libraries(assuan ${links} GPGRT::GPGRT ${links})
set_target_properties(assuan PROPERTIES PUBLIC_HEADER ${CMAKE_BINARY_DIR}/assuan.h)
set_target_properties(assuan PROPERTIES POSITION_INDEPENDENT_CODE TRUE)
set_dll_output_directory(assuan ${CMAKE_BINARY_DIR})
add_library(ASSUAN::ASSUAN ALIAS assuan)

install(TARGETS assuan
		RUNTIME DESTINATION bin
		LIBRARY DESTINATION lib
		ARCHIVE DESTINATION lib
		PUBLIC_HEADER DESTINATION include)

#install(FILES doc/assuan.info DESTINATION share/info)

include(CTest)
if(BUILD_TESTING)

set(tests
version)
if(UNIX)
	list(APPEND tests pipeconnect) #PORT this
endif()
if(USE_DESCRIPTOR_PASSING)
	list(APPEND tests fdpassing)
endif()

add_executable(sock5 tests/socks5.c)
target_link_libraries(sock5 assuan)

foreach(t ${tests})
	add_executable(${t} tests/${t}.c)
	target_link_libraries(${t} assuan)
	add_test(NAME ${t} COMMAND ${t})
	if(WIN32)
		set_tests_properties(${t} PROPERTIES ENVIRONMENT "PATH=${CMAKE_BINARY_DIR}\;${NEW_PATH}")
	endif()
endforeach()

endif()
