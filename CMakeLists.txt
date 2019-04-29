cmake_minimum_required(VERSION 3.4.1)


if(WIN32)
set(PROTOC_EXECUTABLE "C:\\Protobuf\\protoc.exe" CACHE STRING "Path to protoc executable")
set(GRPC_CPP_PLUGIN "C:\\Protobuf\\grpc_cpp_plugin.exe" CACHE STRING "Path to grpc_cpp_plugin executable")
else()
set(PROTOC_EXECUTABLE "/usr/bin/protoc" CACHE STRING "Path to protoc executable")
set(GRPC_CPP_PLUGIN "/usr/bin/grpc_cpp_plugin" CACHE STRING "Path to grpc_cpp_plugin executable")
endif()

set(GRPC_SRC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../third_party/grpc" CACHE BOOL "Directory of grpc source code")

set(GRPC_PROTO_GENS_DIR ${CMAKE_BINARY_DIR}/gens)
file(MAKE_DIRECTORY ${GRPC_PROTO_GENS_DIR})
include_directories(${GRPC_PROTO_GENS_DIR})

function(protobuf_grpc_generate_cpp SRC_FILES HDR_FILES INCLUDE_ROOT)
  if(NOT ARGN)
    message(SEND_ERROR "Error: protobuf_grpc_generate_cpp() called without any proto files")
    return()
  endif()

  set(${SRC_FILES})
  set(${HDR_FILES})
  set(PROTOBUF_INCLUDE_PATH -I ${INCLUDE_ROOT})
  foreach(FIL ${ARGN})
    get_filename_component(ABS_FIL ${FIL} ABSOLUTE)
    get_filename_component(FIL_WE ${FIL} NAME_WE)
    file(RELATIVE_PATH REL_FIL ${INCLUDE_ROOT} ${ABS_FIL})
    get_filename_component(REL_DIR ${REL_FIL} DIRECTORY)
    set(RELFIL_WE "${REL_DIR}/${FIL_WE}")

    message(WARNING "ARGN: ${ARGN}\n FIL: ${FIL}\n ABS_FIL: ${ABS_FIL}\n REL_FIL: ${REL_FIL}\n PROTOBUF_INCLUDE_PATH: ${PROTOBUF_INCLUDE_PATH}")

    list(APPEND ${SRC_FILES} "${GRPC_PROTO_GENS_DIR}/${RELFIL_WE}.pb.cc")
    list(APPEND ${HDR_FILES} "${GRPC_PROTO_GENS_DIR}/${RELFIL_WE}.pb.h")
    list(APPEND ${SRC_FILES} "${GRPC_PROTO_GENS_DIR}/${RELFIL_WE}.grpc.pb.cc")
    list(APPEND ${HDR_FILES} "${GRPC_PROTO_GENS_DIR}/${RELFIL_WE}.grpc.pb.h")

    add_custom_command(
      OUTPUT "${GRPC_PROTO_GENS_DIR}/${RELFIL_WE}.grpc.pb.cc"
             "${GRPC_PROTO_GENS_DIR}/${RELFIL_WE}.grpc.pb.h"
             "${GRPC_PROTO_GENS_DIR}/${RELFIL_WE}.pb.cc"
             "${GRPC_PROTO_GENS_DIR}/${RELFIL_WE}.pb.h"
      COMMAND ${PROTOC_EXECUTABLE}
      ARGS --grpc_out=${GRPC_PROTO_GENS_DIR}
        --cpp_out=${GRPC_PROTO_GENS_DIR}
        --plugin=protoc-gen-grpc=${GRPC_CPP_PLUGIN}
        ${PROTOBUF_INCLUDE_PATH}
        ${REL_FIL}
      WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
      DEPENDS ${PROTOC_EXECUTABLE} ${GRPC_CPP_PLUGIN} ${ABS_FIL} )
  endforeach()

  set_source_files_properties(${${SRC_FILES}} ${${HDR_FILES}} PROPERTIES GENERATED TRUE)
  set(${SRC_FILES} ${${SRC_FILES}} PARENT_SCOPE)
  set(${HDR_FILES} ${${HDR_FILES}} PARENT_SCOPE)
endfunction()

set(PROTO_BASE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src)
file(GLOB_RECURSE PROTO_FILES ${PROTO_BASE_DIR} *.proto)

protobuf_grpc_generate_cpp(
    AZURE_PROTO_SRCS AZURE_PROTO_HDRS ${PROTO_BASE_DIR} ${PROTO_FILES})

add_library(azure_proto_lib
  STATIC ${AZURE_PROTO_HDRS} ${AZURE_PROTO_SRCS})

target_link_libraries(azure_proto_lib
  grpc++_unsecure
  libprotobuf-lite)

  target_include_directories(azure_proto_lib PUBLIC $<BUILD_INTERFACE:${GRPC_PROTO_GENS_DIR}>
  $<INSTALL_INTERFACE:${GRPC_PROTO_GENS_DIR}>)