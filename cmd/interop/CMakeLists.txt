cmake_minimum_required(VERSION 3.12.1)

project(MLSInterop CXX)

###
### Global Config
###

set(APP_NAME "mlspp_client")
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

message("Compiler: " ${CMAKE_CXX_COMPILER_ID})

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU")
  add_compile_options(-Wall -pedantic -Wextra -Werror -Wmissing-declarations)
elseif(MSVC)
  add_compile_options(/W4 /WX)
endif()

###
### Dependencies
###
find_package(OpenSSL 1.1 REQUIRED)

# PkgConfig-based dependencies
find_package(PkgConfig)

pkg_check_modules(PC_PROTOBUF REQUIRED protobuf)
add_library(protobuf::libprotobuf INTERFACE IMPORTED)
set_target_properties(protobuf::libprotobuf PROPERTIES
  INTERFACE_INCLUDE_DIRECTORIES "${PC_PROTOBUF_INCLUDE_DIRS}"
  INTERFACE_LINK_LIBRARIES "${PC_PROTOBUF_LIBRARIES}"
  INTERFACE_COMPILE_OPTIONS "${PC_PROTOBUF_CFLAGS_OTHER}"
  INTERFACE_LINK_OPTIONS "${PC_PROTOBUF_LDFLAGS}")

pkg_check_modules(PC_ZLIB REQUIRED zlib)
add_library(ZLIB::ZLIB INTERFACE IMPORTED)
set_target_properties(ZLIB::ZLIB PROPERTIES
  INTERFACE_INCLUDE_DIRECTORIES "${PC_ZLIB_INCLUDE_DIRS}"
  INTERFACE_LINK_LIBRARIES "${PC_ZLIB_LIBRARIES}"
  INTERFACE_COMPILE_OPTIONS "${PC_ZLIB_CFLAGS_OTHER}"
  INTERFACE_LINK_OPTIONS "${PC_ZLIB_LDFLAGS}")

message("protobuf libs: " ${PC_PROTOBUF_LIBRARIES})
message("protobuf LD flags: " ${PC_PROTOBUF_LDFLAGS})
message("zlib flags: " ${PC_ZLIB_LIBRARIES})
message("zlib LD flags: " ${PC_ZLIB_LDFLAGS})

# gRPC dependencies
#find_package(Protobuf REQUIRED)
find_package(gRPC REQUIRED)
find_package(gflags REQUIRED)

# mlspp
set(CMAKE_EXPORT_PACKAGE_REGISTRY ON)
find_package(MLSPP REQUIRED)

# JSON
find_package(nlohmann_json 3.2 REQUIRED)

###
### Protobuf generation
###

# Get the proto file from the interop repo 
include( ExternalProject )
find_package( Git REQUIRED )
set( MLS_IMPLEMENTATIONS_REPO_URL https://github.com/mlswg/mls-implementations.git )
set_directory_properties( PROPERTIES EP_PREFIX ${CMAKE_BINARY_DIR}/third_party )

ExternalProject_Add(
    mls-interop-extern
    GIT_REPOSITORY ${MLS_IMPLEMENTATIONS_REPO_URL}
    GIT_TAG origin/mlspp
    TIMEOUT 10
    UPDATE_COMMAND ${GIT_EXECUTABLE} pull origin mlspp
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
    LOG_DOWNLOAD ON
   )

ExternalProject_Get_Property(mls-interop-extern SOURCE_DIR)
set(PROTO_PATH ${SOURCE_DIR}/interop/proto CACHE INTERNAL "Folder for .proto data")

# Generate .h and .cc files from the proto file
set(PROTO "mls_client")
get_filename_component(PROTO_FILE "${PROTO_PATH}/${PROTO}.proto" ABSOLUTE)

set(PB_SRC "${CMAKE_CURRENT_BINARY_DIR}/${PROTO}.pb.cc")
set(PB_HDR "${CMAKE_CURRENT_BINARY_DIR}/${PROTO}.pb.h")
set(GRPC_SRC "${CMAKE_CURRENT_BINARY_DIR}/${PROTO}.grpc.pb.cc")
set(GRPC_HDR "${CMAKE_CURRENT_BINARY_DIR}/${PROTO}.grpc.pb.h")
find_program(PROTOBUF_PROTOC protoc)
find_program(GRPC_CPP_PLUGIN_EXECUTABLE grpc_cpp_plugin)

add_custom_command(
      OUTPUT "${PB_SRC}" "${PB_HDR}" "${GRPC_SRC}" "${GRPC_HDR}"
      COMMAND ${PROTOBUF_PROTOC}
      ARGS --grpc_out "${CMAKE_CURRENT_BINARY_DIR}"
        --cpp_out "${CMAKE_CURRENT_BINARY_DIR}"
        -I "${PROTO_PATH}"
        --plugin=protoc-gen-grpc="${GRPC_CPP_PLUGIN_EXECUTABLE}"
        "${PROTO_FILE}"
      DEPENDS mls-interop-extern)

###
### GRPC Executable
###
file(GLOB_RECURSE BIN_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp")
add_executable(${APP_NAME} ${BIN_SOURCES} ${PB_SRC} ${GRPC_SRC})
add_dependencies(${APP_NAME} mls-interop-extern)
target_include_directories(${APP_NAME} PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_link_libraries(${APP_NAME} 
  gflags 
  gRPC::grpc++ protobuf::libprotobuf 
  MLSPP::mlspp MLSPP::mls_vectors MLSPP::tls_syntax)
