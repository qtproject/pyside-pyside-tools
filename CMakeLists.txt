cmake_minimum_required(VERSION 3.1)
project(pyside2-tools)

# Don't display "up-to-date / install" messages when installing, to reduce visual clutter.
if (QUIET_BUILD)
    set(CMAKE_INSTALL_MESSAGE NEVER)
endif()

find_package(Shiboken2 2.0.0 REQUIRED)
find_package(Qt5 REQUIRED Core)
find_package(PySide2 2.0.0 REQUIRED)

set(pyside2_tools_MAJOR_VERSION "2")
set(pyside2_tools_MINOR_VERSION "0")
set(pyside2_tools_MICRO_VERSION "0")
set(pyside2_tools_VERSION "${pyside2_tools_MAJOR_VERSION}.${pyside2_tools_MINOR_VERSION}.${pyside2_tools_MICRO_VERSION}")

option(BUILD_TESTS "Build tests." TRUE)

if(CMAKE_HOST_APPLE)
    set(OSX_USE_LIBCPP "OFF" CACHE BOOL "Explicitly link the libc++ standard library (useful for osx deployment targets lower than 10.9.")
    if(OSX_USE_LIBCPP)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
    endif()
endif()

# UIC stuff
if (NOT PYTHON_SITE_PACKAGES)
    execute_process(
        COMMAND ${SHIBOKEN_PYTHON_INTERPRETER} -c "if True:
            from distutils import sysconfig
            from os.path import sep
            print(sysconfig.get_python_lib(1, 0, prefix='${CMAKE_INSTALL_PREFIX}').replace(sep, '/'))
            "
        OUTPUT_VARIABLE PYTHON_SITE_PACKAGES
    OUTPUT_STRIP_TRAILING_WHITESPACE)
    if (NOT PYTHON_SITE_PACKAGES)
        message(FATAL_ERROR "Could not detect Python module installation directory.")
    endif()
endif()

# Handling .exe extension for Window and the uic/rcc executables
if (WIN32)
    set(EXE_EXT ".exe")
else()
    set(EXE_EXT "")
endif()

set(TOOLS_PATH "${_qt5Core_install_prefix}/bin")
set(UIC_PATH "${TOOLS_PATH}/uic${EXE_EXT}")
set(RCC_PATH "${TOOLS_PATH}/rcc${EXE_EXT}")
if (APPLE)
    set(DESIGNER_PATH "${TOOLS_PATH}/Designer.app")
else()
    set(DESIGNER_PATH "${TOOLS_PATH}/designer${EXE_EXT}")
endif()

install(FILES "${UIC_PATH}"
        DESTINATION bin
        PERMISSIONS
        OWNER_EXECUTE OWNER_WRITE OWNER_READ
        GROUP_EXECUTE GROUP_READ
        WORLD_EXECUTE WORLD_READ)

install(FILES "${RCC_PATH}"
        DESTINATION bin
        PERMISSIONS
        OWNER_EXECUTE OWNER_WRITE OWNER_READ
        GROUP_EXECUTE GROUP_READ
        WORLD_EXECUTE WORLD_READ)

if (EXISTS ${DESIGNER_PATH})
    if (APPLE)
        install(DIRECTORY "${DESIGNER_PATH}"
                DESTINATION bin
                FILE_PERMISSIONS
                OWNER_EXECUTE OWNER_WRITE OWNER_READ
                GROUP_EXECUTE GROUP_READ
                WORLD_EXECUTE WORLD_READ)
    else()
        install(FILES "${DESIGNER_PATH}"
                DESTINATION bin
                PERMISSIONS
                OWNER_EXECUTE OWNER_WRITE OWNER_READ
                GROUP_EXECUTE GROUP_READ
                WORLD_EXECUTE WORLD_READ)
    endif()
endif()

configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake"
               "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
               IMMEDIATE @ONLY)

# When opening super project, prevent redefinition of uninstall target.
if (NOT TARGET uninstall)
    add_custom_target(uninstall "${CMAKE_COMMAND}"
                      -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")
endif()


# Does not work with ninja on Win32.
if (NOT (WIN32 AND CMAKE_GENERATOR STREQUAL "Ninja"))
set(ARCHIVE_NAME ${CMAKE_PROJECT_NAME}-${pyside2_tools_VERSION})
add_custom_target(dist
    COMMAND mkdir -p "${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}" &&
            git log > "${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}/ChangeLog" &&
            git archive --prefix=${ARCHIVE_NAME}/ HEAD --format=tar --output="${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar" &&
            tar -C "${CMAKE_BINARY_DIR}" --owner=root --group=root -r "${ARCHIVE_NAME}/ChangeLog" -f "${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar" &&
            bzip2 -f9 "${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar" &&
            echo "Source package created at ${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar.bz2.\n"
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
endif()

add_subdirectory(pylupdate)

# pyside2-rcc, pyside2-uic, pyside2-designer, shiboken and pyside2-lupdate entrypoints
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/pyside_tool.py
        DESTINATION bin
        PERMISSIONS
        OWNER_EXECUTE OWNER_WRITE OWNER_READ
        GROUP_EXECUTE GROUP_READ
        WORLD_EXECUTE WORLD_READ)

if (BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif ()
