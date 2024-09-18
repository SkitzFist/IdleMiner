cmake_minimum_required(VERSION 3.10)
project(IdleMiner CXX C)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_BUILD_TYPE Release)

include(FetchContent)

# Adding Raylib
set(BUILD_EXAMPLES OFF CACHE BOOL "" FORCE) # don't build the supplied examples
set(BUILD_GAMES    OFF CACHE BOOL "" FORCE) # don't build the supplied example games

if(EMSCRIPTEN)
  set(PLATFORM Web CACHE STRING "" FORCE)
else()
  set(PLATFORM Desktop CACHE STRING "" FORCE)
endif()

FetchContent_Declare(
  raylib 
  GIT_REPOSITORY https://github.com/raysan5/raylib.git 
  GIT_TAG master)
FetchContent_MakeAvailable(raylib)

# Adding our source files
file(GLOB_RECURSE PROJECT_SOURCES CONFIGURE_DEPENDS "${CMAKE_CURRENT_LIST_DIR}/src/*.cpp")
set(PROJECT_INCLUDE "${CMAKE_CURRENT_LIST_DIR}/src/")

# Automatically include all header directories
file(GLOB_RECURSE HEADER_DIRS "${CMAKE_CURRENT_LIST_DIR}/src/*.h")
set(PROJECT_INCLUDE_DIRS "")
foreach(HEADER_FILE ${HEADER_DIRS})
    get_filename_component(HEADER_PATH ${HEADER_FILE} DIRECTORY)
    list(APPEND PROJECT_INCLUDE_DIRS ${HEADER_PATH})
endforeach()
list(REMOVE_DUPLICATES PROJECT_INCLUDE_DIRS)
include_directories(${PROJECT_INCLUDE_DIRS})

# Define the executable
add_executable(${PROJECT_NAME})
target_sources(${PROJECT_NAME} PRIVATE ${PROJECT_SOURCES})
target_include_directories(${PROJECT_NAME} PRIVATE ${PROJECT_INCLUDE})

target_link_libraries(${PROJECT_NAME} raylib)

if(NOT EMSCRIPTEN)
  set(CMAKE_SYSTEM_NAME Windows)
  set(CMAKE_C_COMPILER gcc)
  set(CMAKE_CXX_COMPILER g++)
  set(CMAKE_MAKE_PROGRAM mingw32-make)
endif()

if(EMSCRIPTEN)
  set(CMAKE_EXECUTABLE_SUFFIX ".html")
  target_compile_options(${PROJECT_NAME} PRIVATE -msimd128 -msse4.1 -O3 -pthread)
  set_target_properties(${PROJECT_NAME} PROPERTIES
    SUFFIX ".html"
    LINK_FLAGS "-O3 -s USE_GLFW=3 -sALLOW_MEMORY_GROWTH -DPLATFORM_WEB --shell-file ${CMAKE_SOURCE_DIR}/Shell.html"
  )
else()
  if(MSVC)
    target_compile_options(${PROJECT_NAME} PRIVATE /W4 /O2)
  else()
    target_compile_options(${PROJECT_NAME} PRIVATE -Wno-unused-variable)
    set_target_properties(${PROJECT_NAME} PROPERTIES
      LINK_FLAGS "-O3 -msse2"
    )
  endif()
endif()

# Add command to copy assets folder to the build folder
set(SOURCE_DIR "${CMAKE_SOURCE_DIR}/assets")
set(DEST_DIR "${CMAKE_BINARY_DIR}/assets")

add_custom_command(
    TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
    ${SOURCE_DIR}
    ${DEST_DIR}
    COMMENT "Copying assets directory to the build folder"
)

add_custom_target(copy_assets ALL
    COMMAND ${CMAKE_COMMAND} -E copy_directory
    ${SOURCE_DIR}
    ${DEST_DIR}
    COMMENT "Manually copying assets directory to the build folder"
)

# Add helper makefile to clean cmake cache (it will 're-include' new project folders in src)
add_custom_target(clean-cache
  COMMAND ${CMAKE_COMMAND} -P ${CMAKE_BINARY_DIR}/clean_cache.cmake
  WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
  COMMENT "Cleaning CMake cache"
)

file(WRITE ${CMAKE_BINARY_DIR}/clean_cache.cmake "
  file(REMOVE CMakeCache.txt)
  file(REMOVE_RECURSE CMakeFiles)
")
