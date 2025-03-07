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

# Set BUILD_SHARED_LIBS_PARAM based on platform
if(MSVC)
  set(BUILD_SHARED_LIBS_PARAM ON)  # Shared HiGHS library
else()
  set(BUILD_SHARED_LIBS_PARAM OFF) # Static HiGHS library
endif()

# Download and Build HiGHS (Static)
include(ExternalProject)
ExternalProject_Add(highs_download
  GIT_REPOSITORY https://github.com/ERGO-Code/HiGHS.git
  GIT_TAG "v1.9.0"
  CMAKE_ARGS
    -DCI=OFF
    -DFAST_BUILD=OFF
    -DBUILD_SHARED_LIBS=${BUILD_SHARED_LIBS_PARAM}
    -DCMAKE_BUILD_TYPE=Release
    -DCMAKE_CXX_COMPILER_LAUNCHER=${CMAKE_CXX_COMPILER_LAUNCHER}
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

# Make sure we link against the correct libraries
if(MSVC)
  target_link_libraries(${PROJECT_NAME} 
    ${CMAKE_JS_LIB}
    ${binary_dir}/lib/release/highs.lib
    ${CMAKE_JS_NODELIB_TARGET}
  )
else()
  target_link_libraries(${PROJECT_NAME} 
    ${CMAKE_JS_LIB}
    ${binary_dir}/lib/libhighs.a
  )
endif()

# Make sure highs_download is built before the addon
add_dependencies(${PROJECT_NAME} highs_download)

# Copy highs.dll to the output directory after highs is built
if(MSVC)
  add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E make_directory $<TARGET_FILE_DIR:${PROJECT_NAME}>
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
      ${binary_dir}/bin/release/highs.dll
      $<TARGET_FILE_DIR:${PROJECT_NAME}>/highs.dll
  )
endif()
