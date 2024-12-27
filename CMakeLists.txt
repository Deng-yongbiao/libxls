cmake_minimum_required(VERSION 3.10)

# 项目名称和语言
project(libxls LANGUAGES C CXX)

if(APPLE)
    message(STATUS "Configuring for macos")
    set(CMAKE_OSX_ARCHITECTURES arm64;x86_64)
    set(CMAKE_OSX_DEPLOYMENT_TARGET "10.12" CACHE STRING "Minimum OS X deployment version" FORCE)
    message(STATUS ${CMAKE_OSX_ARCHITECTURES})
    message(STATUS ${CMAKE_OSX_DEPLOYMENT_TARGET})
elseif(WIN32)
    message(STATUS "Configuring for Windows")
endif()

# 设置标准
set(CMAKE_C_STANDARD 99)
set(CMAKE_CXX_STANDARD 17)

# 设置版本号组件
set(LIBXLS_MAJOR_VERSION 1)
set(LIBXLS_MINOR_VERSION 6)
set(LIBXLS_MICRO_VERSION 3)

set(LIBXLS_VERSION "${LIBXLS_MAJOR_VERSION}.${LIBXLS_MINOR_VERSION}.${LIBXLS_MICRO_VERSION}")

# 编译选项
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Wstrict-prototypes -Wno-unused-parameter -pedantic-errors")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")

include(CheckIncludeFile)

# 检测xlocale.h
check_include_file("xlocale.h" HAVE_XLOCALE_H)
# 查找 libiconv 库
find_package(Iconv REQUIRED)
if(ICONV_FOUND)
    set(HAVE_ICONV 1)
else()
    message(FATAL_ERROR "libiconv not found. Please install libiconv and try again.")
endif()
# 生成配置头文件
configure_file(${CMAKE_SOURCE_DIR}/config.h.in ${CMAKE_BINARY_DIR}/config.h)
# 将生成的头文件目录添加到包含路径
include_directories(${CMAKE_BINARY_DIR})

set(HEADER_DIR ${CMAKE_CURRENT_LIST_DIR}/include)
set(SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR}/src)

# 定义头文件
set(HEADER_FILES
        ${HEADER_DIR}/libxls/brdb.c.h
        ${HEADER_DIR}/libxls/brdb.h
        ${HEADER_DIR}/libxls/endian.h
        ${HEADER_DIR}/libxls/locale.h
        ${HEADER_DIR}/libxls/ole.h
        ${HEADER_DIR}/libxls/xlsstruct.h
        ${HEADER_DIR}/libxls/xlstool.h
        ${HEADER_DIR}/libxls/xlstypes.h
        ${HEADER_DIR}/xls.h
)

#定义源文件
set(SOURCE_FILES
        ${SOURCE_DIR}/endian.c
        ${SOURCE_DIR}/locale.c
        ${SOURCE_DIR}/ole.c
        ${SOURCE_DIR}/xls.c
        ${SOURCE_DIR}/xlstool.c
)

option(BUILD_STATIC_LIB "Build static library" ON)

# 添加动态库
add_library(xls SHARED ${SOURCE_FILES} ${HEADER_FILES})
set_target_properties(xls PROPERTIES
        OUTPUT_NAME xls
)
target_link_libraries(xls iconv)  # 链接 iconv 库

if(BUILD_STATIC_LIB)
    message(STATUS "add static lib")
    add_library(xls_static STATIC ${SOURCE_FILES} ${HEADER_FILES})
    set_target_properties(xls_static PROPERTIES
            OUTPUT_NAME xls
    )
    target_link_libraries(xls_static iconv)  # 链接 iconv 库
endif()


########################################################################################################################
#xls c 库的测试
set(TEST_C_DIR ${CMAKE_CURRENT_LIST_DIR}/test)

set(TEST_C_SOURCE_FILES
        ${TEST_C_DIR}/test.c
        ${TEST_C_DIR}/test2.c
)

foreach(TEST_FILE ${TEST_C_SOURCE_FILES})
    get_filename_component(TEST_NAME ${TEST_FILE} NAME_WE)
    add_executable(${TEST_NAME} ${TEST_FILE})
    target_link_libraries(${TEST_NAME} xls)
    add_test(NAME ${TEST_NAME} COMMAND ${TEST_NAME})
endforeach()
########################################################################################################################
##构建xls2csv cmd工具
set(cmd_target xls2csv)
add_executable(${cmd_target} ${CMAKE_CURRENT_LIST_DIR}/src/xls2csv.c)
if(BUILD_STATIC_LIB)
    target_link_libraries(${cmd_target} PRIVATE xls_static)
else ()
    target_link_libraries(${cmd_target} PRIVATE xls)
endif ()
########################################################################################################################
# 构建cpp 的封装库
#set(CPP_DIR ${CMAKE_CURRENT_LIST_DIR}/cplusplus)
#add_library(xlsreader SHARED ${CPP_DIR}/XlsReader.h ${CPP_DIR}/XlsReader.cpp)
#target_link_libraries(xlsreader PRIVATE xls)
#if(BUILD_STATIC_LIB)
#    add_library(xlsreader_static STATIC ${SOURCE_FILES} ${HEADER_FILES})
#    set_target_properties(xlsreader_static PROPERTIES
#            OUTPUT_NAME xlsreader
#    )
#    target_link_libraries(xlsreader_static xls_static)
#endif()
########################################################################################################################
## 构建cpp库 xlsreader 的test
set(CPP_DIR ${CMAKE_CURRENT_LIST_DIR}/cplusplus)
add_executable(test_xlsreader ${CPP_DIR}/main.cpp ${CPP_DIR}/XlsReader.h ${CPP_DIR}/XlsReader.cpp)
if(BUILD_STATIC_LIB)
    target_link_libraries(test_xlsreader PRIVATE xls_static)
else ()
    target_link_libraries(test_xlsreader PRIVATE xls)
endif ()

add_test(NAME test_xlsreader COMMAND test_xlsreader)