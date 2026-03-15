cmake_minimum_required(VERSION 3.15)
project(titan)

option(TITAN_BUILD_INTRINSICS "Build intrinsics targets (may not work on MSVC/Windows)" ON)

# These must be found before LLVM because LLVM's exported targets may link against them
find_package(ZLIB REQUIRED)
find_package(LibXml2 REQUIRED)

# Packages
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
  # Triton and Boost headers: tritonTargets.cmake uses $<INSTALL_INTERFACE:include>
  # which does not propagate correctly on Windows when using MSVC static builds.
  # Pass these via -DTRITON_INCLUDE_DIR and -DBOOST_INCLUDE_DIR from the CI workflow.
  $<$<BOOL:${TRITON_INCLUDE_DIR}>:${TRITON_INCLUDE_DIR}>
  $<$<BOOL:${BOOST_INCLUDE_DIR}>:${BOOST_INCLUDE_DIR}>
)

if(LLVM_DEFINITIONS)
  target_compile_definitions(${PROJECT_NAME} PRIVATE ${LLVM_DEFINITIONS})
endif()

target_compile_features(${PROJECT_NAME} PRIVATE cxx_std_20)

# Link LLVM libs needed by this project
# 注意 LLVM 15 的组件名变化：
#   Scalar  → ScalarOpts（LLVM 15 重命名，LLVMScalar.lib 已不存在）
#   TargetParser 在 LLVM 15 中已合并进 Support，不单独列
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
  # passes & analysis
  Analysis
  Passes
  TransformUtils
  InstCombine
  ScalarOpts       # LLVM 15: 原来的 Scalar 改名为 ScalarOpts
  Vectorize
  IPO
  CFLAliasAnalysis
  Coroutines
  ObjCARC
  # target info (PassBuilder 需要)
  Target
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
