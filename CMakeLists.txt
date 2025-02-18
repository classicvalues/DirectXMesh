# Copyright (c) Microsoft Corporation.
# Licensed under the MIT License.

cmake_minimum_required (VERSION 3.11)

set(DIRECTXMESH_VERSION 1.6.1)

project(DirectXMesh
  VERSION ${DIRECTXMESH_VERSION}
  DESCRIPTION "DirectXMesh geometry Library"
  HOMEPAGE_URL "https://go.microsoft.com/fwlink/?LinkID=324981"
  LANGUAGES CXX)

option(BUILD_TOOLS "Build meshconvert" ON)

# Includes the support for DirectX 12 input layouts
option(BUILD_DX12 "Build with DirectX12 Runtime support" ON)

option(ENABLE_CODE_ANALYSIS "Use Static Code Analysis on build" OFF)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin/CMake")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin/CMake")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin/CMake")

#--- Library
set(LIBRARY_HEADERS
    DirectXMesh/DirectXMesh.h
    DirectXMesh/DirectXMesh.inl)

set(LIBRARY_SOURCES
    DirectXMesh/DirectXMeshP.h
    DirectXMesh/scoped.h
    DirectXMesh/DirectXMeshAdjacency.cpp
    DirectXMesh/DirectXMeshClean.cpp
    DirectXMesh/DirectXMeshGSAdjacency.cpp
    DirectXMesh/DirectXMeshletGenerator.cpp
    DirectXMesh/DirectXMeshNormals.cpp
    DirectXMesh/DirectXMeshOptimize.cpp
    DirectXMesh/DirectXMeshOptimizeLRU.cpp
    DirectXMesh/DirectXMeshOptimizeTVC.cpp
    DirectXMesh/DirectXMeshRemap.cpp
    DirectXMesh/DirectXMeshTangentFrame.cpp
    DirectXMesh/DirectXMeshUtil.cpp
    DirectXMesh/DirectXMeshValidate.cpp
    DirectXMesh/DirectXMeshVBReader.cpp
    DirectXMesh/DirectXMeshVBWriter.cpp
    DirectXMesh/DirectXMeshWeldVertices.cpp)

add_library(${PROJECT_NAME} STATIC ${LIBRARY_SOURCES} ${LIBRARY_HEADERS})

source_group(${PROJECT_NAME} REGULAR_EXPRESSION DirectXMesh/*.*)

target_include_directories(${PROJECT_NAME} PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/DirectXMesh>
  $<INSTALL_INTERFACE:include>)

if (${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.16")
    target_precompile_headers(${PROJECT_NAME} PRIVATE DirectXMesh/DirectXMeshP.h)
endif()

if(MSVC)
    # Use max Warning Level
    string(REPLACE "/W3 " "/Wall " CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})
    string(REPLACE "/W3 " "/Wall " CMAKE_CXX_FLAGS_DEBUG ${CMAKE_CXX_FLAGS_DEBUG})
    string(REPLACE "/W3 " "/Wall " CMAKE_CXX_FLAGS_RELEASE ${CMAKE_CXX_FLAGS_RELEASE})

    # Not using typeid or dynamic_cast, so disable RTTI to save binary size
    string(REPLACE "/GR " "/GR- " CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})
    string(REPLACE "/GR " "/GR- " CMAKE_CXX_FLAGS_DEBUG ${CMAKE_CXX_FLAGS_DEBUG})
    string(REPLACE "/GR " "/GR- " CMAKE_CXX_FLAGS_RELEASE ${CMAKE_CXX_FLAGS_RELEASE})
endif()

if ((NOT WIN32) OR VCPKG_TOOLCHAIN)
    message("INFO: Using VCPKG for DirectX-Headers and DirectXMath.")
    find_package(directx-headers CONFIG REQUIRED)
    find_package(directxmath CONFIG REQUIRED)
    target_link_libraries(${PROJECT_NAME} PRIVATE Microsoft::DirectX-Headers Microsoft::DirectXMath)
    target_compile_definitions(${PROJECT_NAME} PRIVATE USING_DIRECTX_HEADERS)
endif()

#--- Utilities
set(UTILS_HEADERS
    Utilities/FlexibleVertexFormat.h
    Utilities/WaveFrontReader.h)

add_library(Utilities INTERFACE)

source_group(Utilities REGULAR_EXPRESSION Utilities/*.*)

target_include_directories(Utilities INTERFACE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/Utilities>
  $<INSTALL_INTERFACE:utils>)

#--- Package
include(CMakePackageConfigHelpers)

string(TOLOWER ${PROJECT_NAME} PACKAGE_NAME)

write_basic_package_version_file(
  ${PROJECT_BINARY_DIR}/cmake/${PACKAGE_NAME}-config-version.cmake
  VERSION ${DIRECTXMESH_VERSION}
  COMPATIBILITY AnyNewerVersion)

install(TARGETS ${PROJECT_NAME}
  EXPORT ${PROJECT_NAME}-targets
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
  COMPONENT library)

install(TARGETS Utilities
  EXPORT Utilities-targets
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
  COMPONENT utils)

configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/.nuget/${PROJECT_NAME}-config.cmake.in
  ${PROJECT_BINARY_DIR}/cmake/${PACKAGE_NAME}-config.cmake
  INSTALL_DESTINATION cmake/})

install(EXPORT ${PROJECT_NAME}-targets
  FILE ${PROJECT_NAME}-targets.cmake
  NAMESPACE Microsoft::
  DESTINATION cmake/)

install(EXPORT Utilities-targets
  FILE Utilities-targets.cmake
  NAMESPACE Microsoft::DirectXMesh::
  DESTINATION cmake/)

install(FILES ${LIBRARY_HEADERS}
  DESTINATION include)

install(FILES ${UTILS_HEADERS}
  DESTINATION utils)

install(FILES
    ${PROJECT_BINARY_DIR}/cmake/${PACKAGE_NAME}-config.cmake
    ${PROJECT_BINARY_DIR}/cmake/${PACKAGE_NAME}-config-version.cmake
  DESTINATION cmake/)

#--- Command-line tool
if(BUILD_TOOLS AND WIN32 AND (NOT WINDOWS_STORE))
  add_executable(meshconvert
    Meshconvert/Meshconvert.cpp
    Meshconvert/MeshOBJ.cpp
    Meshconvert/Mesh.h
    Meshconvert/Mesh.cpp
    Meshconvert/SDKMesh.h)
  target_include_directories(meshconvert PUBLIC MeshConvert Utilities)
  target_link_libraries(meshconvert ${PROJECT_NAME} version.lib)
  source_group(meshconvert REGULAR_EXPRESSION meshconvert/*.*)

  if (VCPKG_TOOLCHAIN)
    target_link_libraries(meshconvert Microsoft::DirectXMath)
  endif()
endif()
if(MSVC)
    target_compile_options(${PROJECT_NAME} PRIVATE /fp:fast)
    if(BUILD_TOOLS AND WIN32 AND (NOT WINDOWS_STORE))
      target_compile_options(meshconvert PRIVATE /fp:fast)
    endif()

    if(${CMAKE_SIZEOF_VOID_P} EQUAL "4" AND (NOT ${CMAKE_VS_PLATFORM_NAME} MATCHES "arm"))
        target_compile_options(${PROJECT_NAME} PRIVATE /arch:SSE2)
        if(BUILD_TOOLS AND WIN32 AND (NOT WINDOWS_STORE))
          target_compile_options(meshconvert PRIVATE /arch:SSE2)
        endif()
    endif()
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(WarningsLib "-Wpedantic" "-Wextra")
    target_compile_options(${PROJECT_NAME} PRIVATE ${WarningsLib})

    if(BUILD_TOOLS AND WIN32 AND (NOT WINDOWS_STORE))
      set(WarningsEXE ${WarningsLib} "-Wno-c++98-compat" "-Wno-c++98-compat-pedantic" "-Wno-switch" "-Wno-switch-enum" "-Wno-double-promotion" "-Wno-exit-time-destructors" "-Wno-missing-prototypes")
      target_compile_options(meshconvert PRIVATE ${WarningsEXE})
    endif()
endif()
if (CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    target_compile_options(${PROJECT_NAME} PRIVATE /permissive- /JMC- /Zc:__cplusplus)
    if(BUILD_TOOLS AND WIN32 AND (NOT WINDOWS_STORE))
      target_compile_options(meshconvert PRIVATE /permissive- /JMC- /Zc:__cplusplus)
    endif()

   if(ENABLE_CODE_ANALYSIS)
      target_compile_options(${PROJECT_NAME} PRIVATE /analyze)
      if(BUILD_TOOLS AND WIN32 AND (NOT WINDOWS_STORE))
        target_compile_options(meshconvert PRIVATE /analyze)
      endif()
   endif()

    if (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 19.26)
        target_compile_options(${PROJECT_NAME} PRIVATE /Zc:preprocessor /wd5105)
        if(BUILD_TOOLS AND WIN32 AND (NOT WINDOWS_STORE))
          target_compile_options(meshconvert PRIVATE /Zc:preprocessor /wd5105)
        endif()
    endif()

    if(BUILD_TOOLS AND WIN32 AND (NOT WINDOWS_STORE))
      set(WarningsEXE "/wd4061" "/wd4365" "/wd4514" "/wd4625" "/wd4626" "/wd4627" "/wd4668" "/wd4710" "/wd4751" "/wd4820" "/wd5026" "/wd5027" "/wd5039" "/wd5045" "/wd5219")
      target_compile_options(meshconvert PRIVATE ${WarningsEXE})
    endif()
endif()

if(WIN32)
    # Windows 10 is used here to build the DirectX 12 code paths as well as 11
    target_compile_definitions(${PROJECT_NAME} PRIVATE _UNICODE UNICODE)
    if(BUILD_TOOLS AND (NOT WINDOWS_STORE))
      target_compile_definitions(meshconvert PRIVATE _UNICODE UNICODE _WIN32_WINNT=0x0601)
    endif()

    if(BUILD_DX12 OR WINDOWS_STORE)
        target_compile_definitions(${PROJECT_NAME} PRIVATE _WIN32_WINNT=0x0A00)
    else()
        target_compile_definitions(${PROJECT_NAME} PRIVATE _WIN32_WINNT=0x0601)
    endif()
endif()

if(BUILD_TOOLS AND WIN32 AND (NOT WINDOWS_STORE))
  set_property(DIRECTORY PROPERTY VS_STARTUP_PROJECT meshconvert)
endif()
