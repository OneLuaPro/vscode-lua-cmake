# ------------------------------------------------------------------------------
cmake_minimum_required(VERSION 3.31)
project(vscode-lua-cmake C CXX)
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
include("cmake/macros.cmake")

# ------------------------------------------------------------------------------
# Configuration
set(LUA_VSC_VERSION "3.18.2")
if(NOT WIN32)
  message(FATAL_ERROR "This project is currently only supported on Windows.")
endif()

# ------------------------------------------------------------------------------
# Options
# option(SIGN_EXECUTABLE "Code-sign lua-language-server.exe" OFF)
option(SIGN_PACKAGE "Code-sign VSIX package and contents" OFF)

# ------------------------------------------------------------------------------
# Search for tools
find_program(GIT_EXECUTABLE NAMES git REQUIRED)
find_program(VSCE_EXECUTABLE NAMES vsce REQUIRED)
message(STATUS "Found git and vsce.")
# if(SIGN_EXECUTABLE)
#   find_package(signtool)
# endif()
if(SIGN_PACKAGE)
  find_program(SIGN_PACKAGE_EXE NAMES sign REQUIRED)
endif()

# ------------------------------------------------------------------------------
# Setup signing stuff if vsix-package shall be signed
if(SIGN_PACKAGE AND CODE_SIGN_ID)
  # Calculate SHA256 fingerprint (needed by sign.exe)
  get_cert_fingerprint_sha256("${CODE_SIGN_ID}" AUTO_FINGERPRINT)
  if(AUTO_FINGERPRINT)
    set(CODE_SIGN_FINGERPRINT "${AUTO_FINGERPRINT}" CACHE STRING "Auto-detected Fingerprint" FORCE)
    message(STATUS "Auto-detected Certificate Fingerprint: ${CODE_SIGN_FINGERPRINT}")
  else()
    message(FATAL_ERROR "Could not auto-detect fingerprint for Subject: ${CODE_SIGN_ID}")
  endif()
endif()

# ------------------------------------------------------------------------------
# Fetch everythong from https://github.com/LuaLS/vscode-lua
include(FetchContent)
set(PATCH_SRC "${CMAKE_CURRENT_SOURCE_DIR}/patches")
set(RES_SRC "${CMAKE_CURRENT_SOURCE_DIR}/resources")
# Declare fetch
FetchContent_Declare(
  vscode-lua
  GIT_REPOSITORY https://github.com/LuaLS/vscode-lua.git
  GIT_TAG        "v${LUA_VSC_VERSION}"
  SOURCE_DIR     "${CMAKE_BINARY_DIR}/vscode-lua"
  BINARY_DIR     "${CMAKE_BINARY_DIR}/vscode-lua"
)
# Execute fetch
FetchContent_MakeAvailable(vscode-lua)

# ------------------------------------------------------------------------------
# Patch the download
# Setup list of all patches
file(GLOB_RECURSE PATCH_LIST RELATIVE "${PATCH_SRC}" "*.patch")
# Test and apply patches
foreach(PATCH_ENTRY IN LISTS PATCH_LIST)
  # Extract name and dir
  cmake_path(GET PATCH_ENTRY FILENAME PATCH_FILE)
  cmake_path(GET PATCH_ENTRY PARENT_PATH PATCH_DIR)
  if(PATCH_DIR STREQUAL "")
    set(PATCH_DIR ".")
  endif()
  # Test patch. If it can be reverted, it is already applied.
  execute_process(
    COMMAND ${GIT_EXECUTABLE} apply -p1 --reverse --check "${PATCH_SRC}/${PATCH_DIR}/${PATCH_FILE}"
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/vscode-lua/${PATCH_DIR}"
    RESULT_VARIABLE PATCH_ALREADY_APPLIED
    OUTPUT_QUIET
    ERROR_QUIET
  )
  # Apply patch, if not already done
  if(NOT PATCH_ALREADY_APPLIED EQUAL 0)
    # If it's NOT already applied, try to apply it normally.
    message(STATUS "Applying ${PATCH_FILE} ...")
    execute_process(
      COMMAND ${GIT_EXECUTABLE} apply -p1 "${PATCH_SRC}/${PATCH_DIR}/${PATCH_FILE}"
      WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/vscode-lua/${PATCH_DIR}"
      RESULT_VARIABLE PATCH_RESULT
    )
    if(NOT PATCH_RESULT EQUAL 0)
      message(FATAL_ERROR "Patching with ${PATCH_FILE} failed!")
    endif()
  else()
    message(STATUS "Patch ${PATCH_FILE} already applied, skipping.")
  endif()
endforeach()

# ------------------------------------------------------------------------------
# Install OneLuaPro images and icons
file(
  INSTALL "${RES_SRC}/lua-logo-olp-dist-128x128.png"
  DESTINATION "${CMAKE_BINARY_DIR}/vscode-lua/images"
  RENAME "logo.png"
)
file(
  INSTALL "${RES_SRC}/lua-logo-olp-dist.ico"
  DESTINATION "${CMAKE_BINARY_DIR}/vscode-lua/images"
  RENAME "icon.ico"
)
file(
  INSTALL "${RES_SRC}/lua-logo-olp-dist.ico"
  DESTINATION "${CMAKE_BINARY_DIR}/vscode-lua/server/make"
  RENAME "icon.ico"
)

# ------------------------------------------------------------------------------
# Build Server via make.bat
set(SERVER_DIR "${CMAKE_BINARY_DIR}/vscode-lua/server")
set(SERVER_EXE "${SERVER_DIR}/bin/lua-language-server.exe")
add_custom_command(
  OUTPUT "${SERVER_EXE}"
  COMMAND "make.bat"
  WORKING_DIRECTORY "${SERVER_DIR}"
  COMMENT "Building lua-language-server.exe ..."
  VERBATIM
)
# # Optionally sign server binary
# if(SIGN_EXECUTABLE AND signtool_FOUND AND CODE_SIGN_ID)
#   add_custom_command(
#     APPEND
#     OUTPUT "${SERVER_EXE}"
#     COMMAND "${signtool_EXECUTABLE}" sign /n "${CODE_SIGN_ID}" /tr
#     http://ts.ssl.com /td sha256 /fd sha256 "${SERVER_EXE}"
#     COMMENT "Signing lua-language-server.exe with ${CODE_SIGN_ID}..."
#   )
# endif()
# Add target
add_custom_target(build_lua_server
  DEPENDS "${SERVER_EXE}"
)

# ------------------------------------------------------------------------------
# Build Client
add_custom_command(
  OUTPUT "${CMAKE_BINARY_DIR}/vscode-lua/client/webvue/build/index.html"
  COMMAND "buildClient.bat"
  WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/vscode-lua"
  DEPENDS "${SERVER_EXE}"
  COMMENT "Creating lua language client..."
  VERBATIM
)
add_custom_target(build_lua_client
  DEPENDS "${CMAKE_BINARY_DIR}/vscode-lua/client/webvue/build/index.html"
)
add_dependencies(build_lua_client build_lua_server)

# ------------------------------------------------------------------------------
# Create vsix package
set(VSIX_FILE "${CMAKE_BINARY_DIR}/vscode-lua/lua-${LUA_VSC_VERSION}.vsix")
add_custom_command(
  OUTPUT "${VSIX_FILE}"
  COMMAND cmd /c ${VSCE_EXECUTABLE} package
  WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/vscode-lua"
  DEPENDS "${CMAKE_BINARY_DIR}/vscode-lua/client/webvue/build/index.html"
  COMMENT "Creating VSCode package ${VSIX_FILE}..."
  VERBATIM
)
# Optionally sign package
if(SIGN_PACKAGE AND CODE_SIGN_FINGERPRINT AND CODE_SIGN_ID)
  add_custom_command(
    APPEND
    OUTPUT "${VSIX_FILE}"
    COMMAND "${SIGN_PACKAGE_EXE}" code certificate-store ${VSIX_FILE}
    -cfp "${CODE_SIGN_FINGERPRINT}" -pn OneLuaPro -v information
    COMMENT "Signing vsix-package and contents with ${CODE_SIGN_ID}..."
  )
endif()
add_custom_target(package_extension ALL
  DEPENDS "${VSIX_FILE}"
)
add_dependencies(package_extension build_lua_client)

# ------------------------------------------------------------------------------
# Install
install(FILES "${VSIX_FILE}"
  DESTINATION "opt/vscode"
)
# ------------------------------------------------------------------------------
# Notes
#
# mkdir msvc && msvc
# cmake .. -G "Visual Studio 17 2022" -A x64 -DCODE_SIGN_ID="Open Source Developer" -DCMAKE_INSTALL_PREFIX=c:/Apps/OneLuaPro -DSIGN_PACKAGE=ON
# cmake --build . --config Release
# cmake --install .
#
# mkdir intel && cd intel
# cmake -G "Ninja" -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=icx .. -DCMAKE_INSTALL_PREFIX=c:/Apps/OneLuaPro -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded -DCMAKE_C_FLAGS="/DWIN32 /D_WINDOWS /O2 /QxHost /EHsc" -DCMAKE_CXX_FLAGS="/DWIN32 /D_WINDOWS /EHsc /O2 /QxHost /EHsc" -DCMAKE_EXE_LINKER_FLAGS="/Qoption,link,/machine:x64 /Qlibs-static" -DCMAKE_SHARED_LINKER_FLAGS="/Qoption,link,/machine:x64 /Qlibs-static" -DCMAKE_STATIC_LINKER_FLAGS="/Qoption,link,/machine:x64" -DCODE_SIGN_ID="Open Source Developer" -DCMAKE_INSTALL_PREFIX=c:/Apps/OneLuaPro -DSIGN_PACKAGE=ON
# cmake --build .
# cmake --install .
