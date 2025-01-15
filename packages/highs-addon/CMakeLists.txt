cmake_minimum_required(VERSION 3.22)  # Ensure compatible CMake version

cmake_policy(SET CMP0091 NEW)
cmake_policy(SET CMP0042 NEW)

project(highs_addon)

add_definitions(-DNAPI_VERSION=8)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Generate node.lib if needed (Windows-specific)
if(MSVC AND CMAKE_JS_NODELIB_DEF AND CMAKE_JS_NODELIB_TARGET)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MT")  # Static Microsoft C Runtime
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /MT")
  execute_process(COMMAND ${CMAKE_AR} /def:${CMAKE_JS_NODELIB_DEF} /out:${CMAKE_JS_NODELIB_TARGET})
endif() 

# Download and Build HiGHS (Static)
include(ExternalProject)
ExternalProject_Add(highs_download
  GIT_REPOSITORY https://github.com/ERGO-Code/HiGHS.git
  GIT_TAG "66f735e60ce4d4835edaa05a35f2bd4048969c54" # v1.9.0
  CMAKE_ARGS
    -DCI=OFF
    -DFAST_BUILD=OFF
    -DBUILD_SHARED_LIBS=OFF  # Static HiGHS library
    -DCMAKE_BUILD_TYPE=Release
  BUILD_COMMAND ${CMAKE_COMMAND} --build . --config Release --target libhighs
  INSTALL_COMMAND ""
)

# Get HiGHS Build Directories
ExternalProject_Get_Property(highs_download source_dir)
ExternalProject_Get_Property(highs_download binary_dir)

# Build Binding Library
include_directories(${CMAKE_JS_INC} ${source_dir}/src ${binary_dir})
file(GLOB SOURCE_FILES "src/*.cc")
add_library(${PROJECT_NAME} SHARED ${SOURCE_FILES} ${CMAKE_JS_SRC})
set_target_properties(${PROJECT_NAME} PROPERTIES PREFIX "" SUFFIX ".node")

# Set HiGHS Library Path (Static)
if(MSVC)
  set(HIGHS_LIB_PATH "${binary_dir}/lib/Release/highs.lib")
else()
  set(HIGHS_LIB_PATH "${binary_dir}/lib/libhighs.a")
endif()

# Link Libraries (Static HiGHS)
target_link_libraries(${PROJECT_NAME} ${CMAKE_JS_LIB} ${HIGHS_LIB_PATH})

# Build Dependency and Error Handling
add_dependencies(${PROJECT_NAME} highs_download)
