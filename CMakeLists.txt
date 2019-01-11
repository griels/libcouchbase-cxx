cmake_minimum_required(VERSION 3.8)
PROJECT(couchbase++)

INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/include)
IF(NOT MSVC)
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -pedantic")
ENDIF()

SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)
SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
include(ExternalProject)
set(LCB_REPO_PROTOCOL http)
if (LCB_REPO_USERNAME)
    set(LCB_REPO_AT "@")
else(LCB_REPO_USERNAME)
    set(LCB_REPO_AT "")
endif()

set(LCB_REPO ${LCB_REPO_PROTOCOL}://${LCB_REPO_AT}${LCB_REPO_USERNAME}review.couchbase.org/libcouchbase)

SET(LCB_PREFIX ${CMAKE_CURRENT_BINARY_DIR}})
if (WIN32)
    ExternalProject_Add(libcouchbase
            SET(LIBCOUCHBASE_LIBRARY_NAME "libcouchbase.lib")
            SET(LIBCOUCHBASE_DBG_LIBRARY_NAME "libcouchbase_d.lib")
            SET(LIBCOUCHBASE_LIBRARY_FILE "<SOURCE_DIR>/build/lib/Release/${LIBCOUCHBASE_LIBRARY_NAME}")
            SET(LIBCOUCHBASE_DBG_LIBRARY_FILE "<SOURCE_DIR>/dbgbuild/lib/Debug/${LIBCOUCHBASE_DBG_LIBRARY_NAME}")

            PREFIX ${LCB_PREFIX}
            GIT_REPOSITORY ${LCB_REPO}
            GIT_TAG ${_git_rev}

            CONFIGURE_COMMAND ${CMAKE_COMMAND} -E make_directory <SOURCE_DIR>/build
            COMMAND cd <SOURCE_DIR>/build && cmake -DLCB_NO_PLUGINS=1 .. -G "Visual Studio 14 2015 Win64" ${LCB_CMAKE_ARGS}

            BUILD_COMMAND cd <SOURCE_DIR>/build && cmake --build . --config Release ${LCB_CMAKE_ARGS}

            INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/install
            INSTALL_COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/include <INSTALL_DIR>/include
            COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/build/generated <INSTALL_DIR>/include
            COMMAND ${CMAKE_COMMAND} -E make_directory <INSTALL_DIR>/bin/
            COMMAND ${CMAKE_COMMAND} -E make_directory <INSTALL_DIR>/archive/
            COMMAND ${CMAKE_COMMAND} -E make_directory <INSTALL_DIR>/bin/Release
            COMMAND ${CMAKE_COMMAND} -E copy <SOURCE_DIR>/build/bin/Release/libcouchbase.dll <INSTALL_DIR>/bin/Release/
            COMMAND ${CMAKE_COMMAND} -E make_directory <INSTALL_DIR>/archive/Release
            COMMAND ${CMAKE_COMMAND} -E copy <SOURCE_DIR>/build/lib/Release/libcouchbase.lib <INSTALL_DIR>/archive/Release/
            COMMAND ${CMAKE_COMMAND} -E copy <SOURCE_DIR>/build/lib/Release/libcouchbase.exp <INSTALL_DIR>/archive/Release/

            COMMAND ${CMAKE_COMMAND} -E make_directory <SOURCE_DIR>/dbgbuild
            COMMAND cd <SOURCE_DIR>/dbgbuild && cmake .. -DLCB_NO_PLUGINS=1 ${LCB_CMAKE_ARGS}

            COMMAND cd <SOURCE_DIR>/dbgbuild && cmake --build . --config Debug ${LCB_CMAKE_ARGS}

            COMMAND ${CMAKE_COMMAND} -E make_directory <INSTALL_DIR>/bin/Debug
            COMMAND ${CMAKE_COMMAND} -E copy <SOURCE_DIR>/dbgbuild/bin/Debug/libcouchbase_d.dll <INSTALL_DIR>/bin/Debug/
            COMMAND ${CMAKE_COMMAND} -E make_directory <INSTALL_DIR>/archive/Debug
            COMMAND ${CMAKE_COMMAND} -E copy <SOURCE_DIR>/dbgbuild/lib/Debug/libcouchbase_d.lib <INSTALL_DIR>/archive/Debug/
            COMMAND ${CMAKE_COMMAND} -E copy <SOURCE_DIR>/dbgbuild/lib/Debug/libcouchbase_d.exp <INSTALL_DIR>/archive/Debug/


            #COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/CMakeLists_package.txt <INSTALL_DIR>/CMakeLists.txt
            )


else (WIN32)
    if (APPLE)
        SET(LIBCOUCHBASE_LIBRARY_NAME "libcouchbase.dylib")
        SET(LIBCOUCHBASE_DBG_LIBRARY_NAME "libcouchbase.dylib")

    else (APPLE)
        SET(LIBCOUCHBASE_LIBRARY_NAME "libcouchbase.so")
        SET(LIBCOUCHBASE_DBG_LIBRARY_NAME "libcouchbase.so")
    endif (APPLE)
    SET(LIBCOUCHBASE_LIBRARY_FILE "<SOURCE_DIR>/build/lib/${LIBCOUCHBASE_LIBRARY_NAME}")
    SET(LIBCOUCHBASE_DBG_LIBRARY_FILE "<SOURCE_DIR>/dbgbuild/lib/${LIBCOUCHBASE_DBG_LIBRARY_NAME}")


    SET(_parallelism 4)
    ExternalProject_Add(libcouchbase
            PREFIX ${LCB_PREFIX}
            GIT_REPOSITORY ${LCB_REPO}
            GIT_TAG ${_git_rev}

            CONFIGURE_COMMAND ${CMAKE_COMMAND} -E make_directory <SOURCE_DIR>/build
            make_dir <SOURCE_DIR>/dbgbuild
            COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_CURRENT_BINARY_DIR}/install
            COMMAND cd <SOURCE_DIR>/build && cmake .. -DLCB_NO_PLUGINS=1 -DLCB_BUILD_DTRACE=OFF -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>/Release ${LCB_CMAKE_ARGS}
            COMMAND cd <SOURCE_DIR>/dbgbuild && cmake .. -DLCB_NO_PLUGINS=1 -DLCB_BUILD_DTRACE=OFF -DCMAKE_BUILD_TYPE=Debug -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>/Debug ${LCB_CMAKE_ARGS}

            BUILD_IN_SOURCE 1
            BUILD_COMMAND $(MAKE) -j${_parallelism} all install -C build
            COMMAND $(MAKE) -j${_parallelism} all install -C dbgbuild

            INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/install
            INSTALL_COMMAND ${CMAKE_COMMAND} -E rename <INSTALL_DIR>/Release/include <INSTALL_DIR>/include
            COMMAND ${CMAKE_COMMAND} -E make_directory <INSTALL_DIR>/lib
            COMMAND ${CMAKE_COMMAND} -E rename <INSTALL_DIR>/Release/lib <INSTALL_DIR>/lib/Release
            COMMAND ${CMAKE_COMMAND} -E rename <INSTALL_DIR>/Debug/lib <INSTALL_DIR>/lib/Debug
            COMMAND ${CMAKE_COMMAND} -E remove_directory <INSTALL_DIR>/Debug
            COMMAND ${CMAKE_COMMAND} -E remove_directory <INSTALL_DIR>/Release
            #COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/CMakeLists_package.txt <INSTALL_DIR>/CMakeLists.txt
            )

    # OS X-only: Custom post-build step to set the shared library install name.
    if (APPLE)
        ExternalProject_Add_Step(libcouchbase install_name
                COMMAND install_name_tool -id @rpath/libcouchbase.dylib ${LIBCOUCHBASE_LIBRARY_FILE}
                COMMAND install_name_tool -id @rpath/libcouchbase.dylib ${LIBCOUCHBASE_DBG_LIBRARY_FILE}
                DEPENDEES build
                DEPENDERS install
                )
    endif (APPLE)
endif (WIN32)

IF(WIN32)
    SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)
    ADD_DEFINITIONS(-D_CRT_SECURE_NO_WARNINGS)
ELSE()
    SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
ENDIF()

ADD_EXECUTABLE(example ${PROJECT_SOURCE_DIR}/example.cpp)
TARGET_LINK_LIBRARIES(example couchbase)
add_dependencies(example libcouchbase)

ENABLE_TESTING()
ADD_SUBDIRECTORY(tests)
