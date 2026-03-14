cmake_minimum_required(VERSION 3.15)

project(titan)

option(TITAN_BUILD_INTRINSICS "Build intrinsics targets (may not work on MSVC/Windows)" ON)

# Packages
# Packages
find_package(ZLIB REQUIRED)
find_package(LibXml2 REQUIRED)

find_package(fmt CONFIG REQUIRED)
find_package(LLVM 15.0 CONFIG REQUIRED)
find_package(triton CONFIG REQUIRED)
find_package(range-v3 CONFIG REQUIRED)

file(GLOB_RECURSE TITAN_SOURCES  CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp")
file(GLOB_RECURSE TITAN_INCLUDES CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/*.hpp")
source_group(TREE ${PROJECT_SOURCE_DIR} FILES ${TITAN_SOURCES} ${TITAN_INCLUDES})

add_executable(${PROJECT_NAME} ${TITAN_SOURCES} ${TITAN_INCLUDES})

target_include_directories(${PROJECT_NAME} PRIVATE
  "src/"
  ${LLVM_INCLUDE_DIRS}
)

if(LLVM_DEFINITIONS)
  target_compile_definitions(${PROJECT_NAME} PRIVATE ${LLVM_DEFINITIONS})
endif()

target_compile_features(${PROJECT_NAME} PRIVATE cxx_std_20)

# LLVM link (components -> libraries)
# This is required to resolve symbols from llvm::cl, llvm::Support, llvm::Object, etc.
llvm_map_components_to_libnames(LLVM_LIBS
  Support
  Option
  Object
  BitReader
  Core
  IRReader
  MC
  MCParser
  Remarks
)

target_link_libraries(${PROJECT_NAME} PRIVATE
  fmt::fmt
  range-v3::range-v3
  triton::triton
  ${LLVM_LIBS}
)

if(TITAN_BUILD_INTRINSICS)
  add_subdirectory(intrinsics)
endif()
