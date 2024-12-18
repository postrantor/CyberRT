cmake_minimum_required(VERSION 3.8)
project(cyber VERSION 9.0.0)
set(TARGET_NAME ${PROJECT_NAME})

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")

enable_language(ASM)

include(CMakePackageConfigHelpers)
include(GNUInstallDirs)

# ccache
find_program(CCACHE_FOUND ccache)

if(CCACHE_FOUND)
  message(STATUS "### use ccache")
  set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
endif()

if(${CMAKE_SYSTEM_PROCESSOR} STREQUAL "x86_64")
  message("### SYSTEM_PROCESSOR: ${CMAKE_SYSTEM_PROCESSOR} ${CROUTINE_FILE}")
  set(CROUTINE_FILE "cyber/croutine/detail/swap_x86_64.S")
else()
  message("### SYSTEM_PROCESSOR: ${CMAKE_SYSTEM_PROCESSOR} ${CROUTINE_FILE}")
  set(CROUTINE_FILE "cyber/croutine/detail/swap_aarch64.S")
endif()

find_package(fastrtps REQUIRED)
find_package(fastcdr REQUIRED)
find_package(glog REQUIRED)
find_package(Threads REQUIRED)
find_package(PkgConfig REQUIRED)
find_package(Protobuf REQUIRED)
find_package(tinyxml2 REQUIRED)

pkg_check_modules(Python REQUIRED python3)
pkg_check_modules(Uuid REQUIRED uuid)
pkg_check_modules(NlohmannJson REQUIRED nlohmann_json)

set(Python_VERSION "${Python_VERSION}")
set(CYBER_PYTHON_PATH "lib/python${Python_VERSION}/site-packages/cyber/python")
set(CYBER_PYTHON_INTERNAL_PATH "${CYBER_PYTHON_PATH}/internal")
set(CYBER_PYTHON_PROTOBUF_PATH "lib/python${Python_VERSION}/site-packages/cyber/proto")
set(CYBER_MSGS_CACHE ON)

include_directories(
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${Protobuf_INCLUDE_DIRS}
  ${NlohmannJson_INCLUDE_DIRS}
  ${Python_INCLUDE_DIRS}
)

file(GLOB CYBER_PROTO_FILES
  "${CMAKE_CURRENT_SOURCE_DIR}/modules/common_msgs/**/*.proto"
  "${CMAKE_CURRENT_SOURCE_DIR}/cyber/proto/*.proto"
  "${CMAKE_CURRENT_SOURCE_DIR}/cyber/examples/proto/*.proto"
)

if(${CYBER_MSGS_CACHE})
  set(CYBER_MSGS_CACHE OFF CACHE BOOL "generate cyber proto msgs flag." FORCE)

  foreach(FIL ${CYBER_PROTO_FILES})
    message("# generate proto file: ${FIL}")
    get_filename_component(FIL_WE ${FIL} NAME_WE)
    string(REGEX REPLACE ".+/(.+)\\..*" "\\1" FILE_NAME ${FIL})
    string(REGEX REPLACE "(.+)\\${FILE_NAME}.*" "\\1" FILE_PATH ${FIL})
    execute_process(
      COMMAND
      ${PROTOBUF_PROTOC_EXECUTABLE}
      -I${CMAKE_CURRENT_SOURCE_DIR}
      --cpp_out=${CMAKE_CURRENT_SOURCE_DIR}
      --python_out=${CMAKE_CURRENT_SOURCE_DIR}
      ${FIL}
    )
  endforeach()
endif()

file(GLOB PROTO_SRCS
  "cyber/proto/*.pb.cc"
  "cyber/proto/*.pb.h"
  "cyber/examples/proto/*.pb.cc"
  "cyber/examples/proto/*.pb.h"
  "modules/common_msgs/**/*.pb.cc"
  "modules/common_msgs/**/*.pb.h"
)

file(GLOB PROJECT_SRCS
  "cyber/base/*.cc"
  "cyber/blocker/*.cc"
  "cyber/class_loader/*.cc"
  "cyber/class_loader/*/*.cc"
  "cyber/common/*.cc"
  "cyber/component/*.cc"
  "cyber/croutine/*.cc"
  "cyber/croutine/*/*.cc"
  "cyber/data/*.cc"
  "cyber/event/*.cc"
  "cyber/io/*.cc"
  "cyber/logger/*.cc"
  "cyber/message/*.cc"
  "cyber/node/*.cc"
  "cyber/parameter/*.cc"
  "cyber/plugin_manager/*.cc"
  "cyber/record/*.cc"
  "cyber/record/*/*.cc"
  "cyber/scheduler/*.cc"
  "cyber/scheduler/*/*.cc"
  "cyber/service/*.cc"
  "cyber/service_discovery/*.cc"
  "cyber/service_discovery/*/*.cc"
  "cyber/task/*.cc"
  "cyber/time/*.cc"
  "cyber/timer/*.cc"
  "cyber/transport/*.cc"
  "cyber/transport/*/*.cc"
  "cyber/sysmo/*.cc"
  "cyber/*.cc"
  ${CROUTINE_FILE}
)

list(FILTER PROJECT_SRCS EXCLUDE REGEX .*test[.]cc)
list(FILTER PROJECT_SRCS EXCLUDE REGEX .*/cyber/class_loader/test/*)
list(REMOVE_ITEM PROJECT_SRCS "${CMAKE_CURRENT_SOURCE_DIR}/cyber/class_loader/shared_library/sample.cc")

add_library(${PROJECT_NAME} SHARED
  ${PROTO_SRCS}
  ${PROJECT_SRCS}
)

target_link_libraries(${PROJECT_NAME}
  ${Uuid_LIBRARIES}
  ${Protobuf_LIBRARIES}
  tinyxml2::tinyxml2
  glog::glog
  fastrtps
  fastcdr
  atomic
  dl
  rt
)

add_subdirectory(cyber/mainboard)
add_subdirectory(cyber/python)
add_subdirectory(cyber/tools)
add_subdirectory(cyber/examples)

# CPack
include(InstallRequiredSystemLibraries)
set(CPACK_GENERATOR DEB)
set(CPACK_SOURCE_GENERATOR TGZ ZIP)
set(CPACK_PACKAGE_NAME "${PROJECT_NAME}")
set(CPACK_PACKAGE_CONTACT "zhiqi.jia@hirain.com")
set(CPACK_PACKAGE_VERSION "${PROJECT_VERSION}")
set(CPACK_PACKAGE_VERSION_MAJOR "${PROJECT_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${PROJECT_VERSION_MINOR}")
set(CPACK_PACKAGE_VERSION_PATCH "${PROJECT_VERSION_PATCH}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Hirain GaeaRT")
set(CPACK_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}")
set(CPACK_OUTPUT_FILE_PREFIX "packages")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_MONOLITHIC_INSTALL TRUE)
set(CPACK_PACKAGING_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX})
set(CPACK_RPM_PACKAGE_GROUP "Development/Libraries")
set(CPACK_RPM_PACKAGE_LICENSE "MIT")
set(CPACK_DEBIAN_PACKAGE_HOMEPAGE "https://gerrit-crdc.hirain.com/admin/repos/huixi/r1sdk/GaeaRT")
set(CPACK_PACKAGE_FILE_NAME "lib${CPACK_PACKAGE_NAME}")
set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_FILE_NAME}_${CPACK_PACKAGE_VERSION}_${CMAKE_SYSTEM_PROCESSOR}")

include(CPack)

# package
configure_file(
  "cmake/${PROJECT_NAME}.pc.in"
  "${PROJECT_BINARY_DIR}/pkgconfig/${PROJECT_NAME}.pc" @ONLY
)

configure_package_config_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake/${PROJECT_NAME}-config.cmake.in"
  "cmake/${PROJECT_NAME}-config.cmake"
  INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
  NO_CHECK_REQUIRED_COMPONENTS_MACRO
  PATH_VARS CMAKE_INSTALL_INCLUDEDIR CMAKE_INSTALL_LIBDIR PROJECT_NAME
)

write_basic_package_version_file(
  cmake/${PROJECT_NAME}-config-version.cmake
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY AnyNewerVersion
)

install(
  FILES "${PROJECT_BINARY_DIR}/pkgconfig/${PROJECT_NAME}.pc"
  DESTINATION "${CMAKE_INSTALL_DATADIR}/pkgconfig"
)

install(
  FILES
  "${PROJECT_BINARY_DIR}/cmake/${PROJECT_NAME}-config.cmake"
  "${PROJECT_BINARY_DIR}/cmake/${PROJECT_NAME}-config-version.cmake"
  DESTINATION "${CMAKE_INSTALL_DATADIR}/cmake/${PROJECT_NAME}"
)

install(
  EXPORT ${PROJECT_NAME}-targets
  DESTINATION "${CMAKE_INSTALL_DATADIR}/cmake/${PROJECT_NAME}"
)

# include header
install(
  DIRECTORY cyber/
  DESTINATION include/cyber/
  FILES_MATCHING
  PATTERN "*.h"
)

# apollo conf
file(
  COPY cyber/conf/
  DESTINATION ${CMAKE_BINARY_DIR}/share/conf/
)
install(
  DIRECTORY ${CMAKE_BINARY_DIR}/share/conf/
  DESTINATION share/conf/
)

# setup.zsh
configure_file("setup.bash.in" "${CMAKE_BINARY_DIR}/setup.bash" @ONLY)
configure_file("setup.zsh.in" "${CMAKE_BINARY_DIR}/setup.zsh" @ONLY)
install(
  FILES
  ${CMAKE_BINARY_DIR}/setup.zsh
  ${CMAKE_BINARY_DIR}/setup.bash
  DESTINATION .
)

# cyber python api
file(
  COPY cyber/python/
  DESTINATION ${CMAKE_BINARY_DIR}/${CYBER_PYTHON_PATH}
)
install(
  DIRECTORY ${CMAKE_BINARY_DIR}/${CYBER_PYTHON_PATH}/
  DESTINATION ${CYBER_PYTHON_PATH}
  FILES_MATCHING
  PATTERN "*.py"
)

# cyber python protobuf
file(WRITE ${CMAKE_BINARY_DIR}/${CYBER_PYTHON_PROTOBUF_PATH}/__init__.py "import os\n")
file(APPEND ${CMAKE_BINARY_DIR}/${CYBER_PYTHON_PROTOBUF_PATH}/__init__.py "import sys\n")
file(APPEND ${CMAKE_BINARY_DIR}/${CYBER_PYTHON_PROTOBUF_PATH}/__init__.py "sys.path.insert(0, os.path.abspath(os.path.dirname(__file__)))\n")
file(
  COPY cyber/proto/
  DESTINATION ${CMAKE_BINARY_DIR}/${CYBER_PYTHON_PROTOBUF_PATH}
  FILES_MATCHING
  PATTERN "*.py"
)

install(
  DIRECTORY ${CMAKE_BINARY_DIR}/${CYBER_PYTHON_PROTOBUF_PATH}/
  DESTINATION ${CYBER_PYTHON_PROTOBUF_PATH}
)

install(
  TARGETS ${PROJECT_NAME}
  EXPORT ${PROJECT_NAME}-targets
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)
