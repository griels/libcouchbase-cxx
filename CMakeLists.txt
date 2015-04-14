CMAKE_MINIMUM_REQUIRED(VERSION 2.8.9)
PROJECT(couchbase++)

INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/include)
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")

SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)
SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
IF(WIN32)
    SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)
    ADD_DEFINITIONS(-D_CRT_SECURE_NO_WARNINGS)
ELSE()
    SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
ENDIF()

ADD_EXECUTABLE(example ${PROJECT_SOURCE_DIR}/example.cpp)
TARGET_LINK_LIBRARIES(example couchbase)