# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# License); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# AS IS BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#
# Copyright (c) 2021, OPEN AI LAB
# Author: xlchen@openailab.com
#

# generate tengine header file
FILE (MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/tengine)
FILE (COPY ${CMAKE_SOURCE_DIR}/source/api/c_api.h DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/tengine)

# add protobuf
find_package(Protobuf 3.10.0)
include_directories(${Protobuf_INCLUDE_DIR})
if (${Protobuf_VERSION} VERSION_LESS "3.10.0")
    message(WARNING "protobuf version is ${Protobuf_VERSION}, need 3.10.0+, may make failed.")
endif()

# ONNX
FILE(GLOB_RECURSE ONNX_SERIALIZER_SRCS "${CMAKE_CURRENT_SOURCE_DIR}/onnx/*.cpp")
# generate pb.cc 
set(ONNX_PROTO_SRC          ${CMAKE_CURRENT_BINARY_DIR}/onnx/onnx.pb.cc)
set(ONNX_PROTO_PATH         ${CMAKE_CURRENT_SOURCE_DIR}/onnx)
set(ONNX_PROTO_OUT_PATH     ${CMAKE_CURRENT_BINARY_DIR}/onnx)

ADD_CUSTOM_COMMAND(OUTPUT   ${ONNX_PROTO_SRC}
                   COMMAND mkdir -p ${ONNX_PROTO_OUT_PATH}
                   COMMAND ${Protobuf_PROTOC_EXECUTABLE} --cpp_out=${ONNX_PROTO_OUT_PATH} --proto_path=${ONNX_PROTO_PATH} ${ONNX_PROTO_PATH}/onnx.proto
                   DEPENDS ${ONNX_PROTO_PATH}/onnx.proto)

ADD_CUSTOM_TARGET(ONNX_SERIALIZER_TARGET DEPENDS ${ONNX_PROTO_SRC})
include_directories(${ONNX_PROTO_OUT_PATH})
list(APPEND ONNX_SERIALIZER_SRCS ${ONNX_PROTO_SRC})

# SAVE GRAPH
FILE(GLOB_RECURSE SAVE_GRAPH_SRCS "${CMAKE_CURRENT_SOURCE_DIR}/utils/save_graph/*.cpp" "${CMAKE_CURRENT_SOURCE_DIR}/utils/save_graph/*.c")

# GRAPH OPTIMIZER
FILE(GLOB_RECURSE GRAPH_OPT_SRCS "${CMAKE_CURRENT_SOURCE_DIR}/utils/graph_optimizer/*.cpp")

# CONVERT TOOL
FILE(GLOB_RECURSE CONVERT_TOOL_SRCS "${CMAKE_CURRENT_SOURCE_DIR}/convert_tool.cpp")
list(APPEND CONVERT_TOOL_SRCS ${ONNX_SERIALIZER_SRCS})
list(APPEND CONVERT_TOOL_SRCS ${SAVE_GRAPH_SRCS})
list(APPEND CONVERT_TOOL_SRCS ${GRAPH_OPT_SRCS})

add_executable(convert_tool ${CONVERT_TOOL_SRCS})
target_link_libraries(convert_tool ${CMAKE_PROJECT_NAME}-static pthread dl m)
target_link_libraries(convert_tool ${Protobuf_LIBRARIES})
TARGET_INCLUDE_DIRECTORIES (convert_tool PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
TARGET_INCLUDE_DIRECTORIES (convert_tool PRIVATE "${CMAKE_SOURCE_DIR}/source")
TARGET_INCLUDE_DIRECTORIES (convert_tool PRIVATE "${CMAKE_BINARY_DIR}/source")
TARGET_INCLUDE_DIRECTORIES (convert_tool PRIVATE "${CMAKE_SOURCE_DIR}/source/operator/prototype")
install(TARGETS convert_tool DESTINATION bin)

# add to a virtual project group
SET_PROPERTY(TARGET ONNX_SERIALIZER_TARGET PROPERTY FOLDER "tools/convert_tool")
SET_PROPERTY(TARGET convert_tool PROPERTY FOLDER "tools/convert_tool")
