cmake_minimum_required(VERSION 3.5)

project(libssh2-cmake LANGUAGES CXX C)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if (NOT UNIX)
    message(FATAL_ERROR "unsupporsted ${CMAKE_SYSTEM_NAME}")
endif()

set (install_prefix "${CMAKE_CURRENT_BINARY_DIR}/local")

set(THIRDPARTY_LIBSSH2_PREFIX ${install_prefix} PARENT_SCOPE)
set(HAS_THIRDPARTY_LIBSSH2 ON PARENT_SCOPE)

include(ExternalProject)

function(libssh2Main)

    add_library(thirdparty_libssh2 INTERFACE)
    add_library(thirdparty_libssh2_static STATIC IMPORTED GLOBAL)
    if (NOT CMD_MAKE)
        message(STATUS "[libssh2] set CMD_MAKE is make.")
        set(CMD_MAKE make)
    endif()

    message(STATUS "[libssh2] add external project: libssh2")

    # 依赖库 https://curl.se/docs/libs.html
    set(configure_command
        "${CMAKE_COMMAND}" -E env "CC=${CMAKE_C_COMPILER}" "AR=${CMAKE_AR}" "LIBS=-lpthread -ldl"
        perl ./configure
        --with-sysroot=${CMAKE_SYSROOT}
        --enable-static
        --with-libssl-prefix=${THIRDPARTY_OPENSSL_PREFIX}
        "--prefix=${install_prefix}"
        "CFLAGS=-fPIC"
        "CPPFLAGS=-fPIC"
        )

    ExternalProject_Add(
        libssh2
        URL "https://www.libssh2.org/download/libssh2-1.10.0.tar.gz"
        URL_HASH "SHA256=2d64e90f3ded394b91d3a2e774ca203a4179f69aebee03003e5a6fa621e41d51"
        CONFIGURE_COMMAND ${configure_command}
        BUILD_COMMAND ${CMD_MAKE}
        BUILD_IN_SOURCE 1
        INSTALL_COMMAND ${CMD_MAKE} install
        UPDATE_COMMAND ""
        )
    add_dependencies(thirdparty_libssh2_static libssh2)

    set_target_properties(thirdparty_libssh2_static PROPERTIES IMPORTED_LOCATION
        "${install_prefix}/lib/libssh2.a"
        )

    target_link_libraries(thirdparty_libssh2 INTERFACE thirdparty_libssh2_static)
    if (CMAKE_SYSTEM_NAME MATCHES "Darwin")
        target_link_libraries(thirdparty_libssh2 INTERFACE "-framework SystemConfiguration" dl z)
    endif()
    target_include_directories(thirdparty_libssh2 INTERFACE "${install_prefix}/include")

endfunction()

libssh2Main()
