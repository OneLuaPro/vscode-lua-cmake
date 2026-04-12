# ------------------------------------------------------------------------------
cmake_minimum_required(VERSION 3.31)
cmake_policy(SET CMP0114 NEW)

project(vscode-lua-cmake C CXX)

include(ExternalProject)

# ------------------------------------------------------------------------------
# Configuration
set(LUA_VSC_VERSION "3.18.1")
if(NOT WIN32)
  message(FATAL_ERROR "This project is currently only supported on Windows.")
endif()
set(CODE_SIGN_ID "" CACHE STRING "Certificate ID for code signing")
set(CODE_SIGN_FINGERPRINT "" CACHE STRING
  "Certificate fingerprint for vsix package signing")
option(SIGN_VSIX_PACKAGE "Sign generated vsix-package" OFF)

# ------------------------------------------------------------------------------
# Search for tools
find_program(GIT_EXECUTABLE NAMES git REQUIRED)
find_program(NPM_EXECUTABLE NAMES npm REQUIRED)
find_program(VSCE_EXECUTABLE NAMES vsce REQUIRED)

# ------------------------------------------------------------------------------
# Setup signing stuff
macro(get_cert_fingerprint_sha256 SUBJECT_PATTERN RESULT_VAR)
  set(PS_CMD "$cert = Get-ChildItem Cert:\\CurrentUser\\My | Where-Object { $_.Subject -like '*${SUBJECT_PATTERN}*' } | Select-Object -First 1; if($cert) { $hash = [System.Security.Cryptography.SHA256]::Create().ComputeHash($cert.RawData); $hash | ForEach-Object { '{0:X2}' -f $_ } | Write-Host -NoNewline }")    
  execute_process(
    COMMAND powershell -NoProfile -NonInteractive -Command "${PS_CMD}"
    OUTPUT_VARIABLE ${RESULT_VAR}
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET
    COMMAND_ECHO NONE	# for debug: STDOUT
  )
endmacro()
if(CODE_SIGN_ID)
  if(NOT signtool_FOUND)
    message(WARNING
      "CODE_SIGN_ID set but signtool_EXECUTABLE not defined by superbuild.")
  endif()
  if(SIGN_VSIX_PACKAGE)
    find_program(VSIX_SIGN_TOOL NAMES sign REQUIRED)
    # Calculate SHA256 fingerprint (needed by sign.exe)
    get_cert_fingerprint_sha256("${CODE_SIGN_ID}" AUTO_FINGERPRINT)
    if(AUTO_FINGERPRINT)
      set(CODE_SIGN_FINGERPRINT "${AUTO_FINGERPRINT}" CACHE STRING "Auto-detected Fingerprint" FORCE)
      message(STATUS "Auto-detected Certificate Fingerprint: ${CODE_SIGN_FINGERPRINT}")
    else()
      message(WARNING "Could not auto-detect fingerprint for Subject: ${CODE_SIGN_ID}")
    endif()
  endif()
endif()

# ------------------------------------------------------------------------------
# Paths
set(VSC_LUA_SRC "${CMAKE_BINARY_DIR}/vscode-lua")
set(PATCH_DIR "${CMAKE_CURRENT_SOURCE_DIR}/patches")
set(VSIX_NAME "lua-${LUA_VSC_VERSION}.vsix")

# ------------------------------------------------------------------------------
ExternalProject_Add(vscode_lua_ext
  GIT_REPOSITORY      "https://github.com/LuaLS/vscode-lua.git"
  GIT_TAG             "v${LUA_VSC_VERSION}"
  GIT_SUBMODULES_RECURSE ON
  SOURCE_DIR          "${VSC_LUA_SRC}"

  # We set BINARY_DIR to SOURCE_DIR so that all commands 
  # run directly inside the cloned repository root.
  BINARY_DIR          "${VSC_LUA_SRC}/server"

  PATCH_COMMAND
  "${GIT_EXECUTABLE}" checkout . --recurse-submodules &&
  #
  ${CMAKE_COMMAND} -E copy_directory "${PATCH_DIR}" "${VSC_LUA_SRC}" &&
  #
  ${CMAKE_COMMAND} -E remove -f "${VSC_LUA_SRC}/images/logo.png" &&
  ${CMAKE_COMMAND} -E copy
  "${CMAKE_CURRENT_SOURCE_DIR}/resources/lua-logo-olp-dist-128x128.png"
  "${VSC_LUA_SRC}/images/logo.png" &&
  #
  ${CMAKE_COMMAND} -E remove -f "${VSC_LUA_SRC}/images/icon.ico" &&
  ${CMAKE_COMMAND} -E copy
  "${CMAKE_CURRENT_SOURCE_DIR}/resources/lua-logo-olp-dist.ico"
  "${VSC_LUA_SRC}/images/icon.ico" &&
  #
  ${CMAKE_COMMAND} -E remove -f "${VSC_LUA_SRC}/server/make/icon.ico" &&
  ${CMAKE_COMMAND} -E copy
  "${CMAKE_CURRENT_SOURCE_DIR}/resources/lua-logo-olp-dist.ico"
  "${VSC_LUA_SRC}/server/make/icon.ico" &&
  #
  "${GIT_EXECUTABLE}" apply -p1 client.patch &&
  "${GIT_EXECUTABLE}" apply -p1 server.patch --directory=server/ &&
  "${GIT_EXECUTABLE}" apply -p1 luamake.patch --directory=server/3rd/luamake/
  
  CONFIGURE_COMMAND   ""
  BUILD_COMMAND       "make.bat"
  INSTALL_COMMAND     ""
)

# ------------------------------------------------------------------------------
# --- OPTIONAL STEP: SIGN SERVER ---
if(CODE_SIGN_ID AND signtool_FOUND)
  file(TO_NATIVE_PATH "${signtool_EXECUTABLE}" NATIVE_SIGNTOOL)
  ExternalProject_Add_Step(vscode_lua_ext sign_server
    COMMAND "${NATIVE_SIGNTOOL}" sign /n "${CODE_SIGN_ID}" /tr http://ts.ssl.com /td sha256 /fd sha256 bin/lua-language-server.exe
    DEPENDEES build
    WORKING_DIRECTORY "${VSC_LUA_SRC}/server"
  )
endif()

# ------------------------------------------------------------------------------
# --- PHASE: BUILD CLIENT ---
if(CODE_SIGN_ID AND signtool_FOUND)
  set(CLIENT_BUILD_DEP sign_server)
else()
  set(CLIENT_BUILD_DEP build)
endif()
ExternalProject_Add_Step(vscode_lua_ext build_client
  COMMAND "buildClient.bat"
    DEPENDEES ${CLIENT_BUILD_DEP}
  # If sign_server exists, it will run before this due to BEFORE/DEPENDEES logic
  WORKING_DIRECTORY "${VSC_LUA_SRC}"
)

# ------------------------------------------------------------------------------
# --- PHASE: PACKAGE ---
ExternalProject_Add_Step(vscode_lua_ext package_client
  COMMAND "${VSCE_EXECUTABLE}" package
  DEPENDEES build_client
  WORKING_DIRECTORY "${VSC_LUA_SRC}"
  BYPRODUCTS "${VSC_LUA_SRC}/${VSIX_NAME}"
)

set(STEP_AFTER_PACKAGE package_client)

# ------------------------------------------------------------------------------
# --- OPTIONAL STEP: SIGN PACKAGE ---
if(CODE_SIGN_ID AND CODE_SIGN_FINGERPRINT)
  ExternalProject_Add_Step(vscode_lua_ext sign_package
    COMMAND "${VSIX_SIGN_TOOL}" code certificate-store -cfp ${CODE_SIGN_FINGERPRINT}
    -u "http://certum.pl" "${VSIX_NAME}"
    DEPENDEES package_client
    WORKING_DIRECTORY "${VSC_LUA_SRC}"
  )
  set(STEP_AFTER_PACKAGE sign_package)
else()
  message(STATUS "Will not sign vsix-package.")
endif()

# This ensures that 'cmake --build .' actually runs all steps up to the final VSIX
ExternalProject_Add_StepTargets(vscode_lua_ext ${STEP_AFTER_PACKAGE})
add_dependencies(vscode_lua_ext vscode_lua_ext-${STEP_AFTER_PACKAGE})

# ------------------------------------------------------------------------------
# --- Install ---
install(FILES "${VSC_LUA_SRC}/${VSIX_NAME}" 
  DESTINATION "opt/vscode")

# Notes
# ------------------------------------------------------------------------------
# mkdir intel && cd intel
# cmake -G "Ninja" -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=icx .. -DCMAKE_INSTALL_PREFIX=c:/Apps/OneLuaPro -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded -DCMAKE_C_FLAGS="/DWIN32 /D_WINDOWS /O2 /QxHost /EHsc" -DCMAKE_CXX_FLAGS="/DWIN32 /D_WINDOWS /EHsc /O2 /QxHost /EHsc" -DCMAKE_EXE_LINKER_FLAGS="/Qoption,link,/machine:x64 /Qlibs-static" -DCMAKE_SHARED_LINKER_FLAGS="/Qoption,link,/machine:x64 /Qlibs-static" -DCMAKE_STATIC_LINKER_FLAGS="/Qoption,link,/machine:x64" -DCODE_SIGN_ID="Open Source Developer" -Dsigntool_FOUND=TRUE -Dsigntool_EXECUTABLE="C:/Program Files (x86)/Windows Kits/10/bin/10.0.22621.0/x64/signtool.exe" -DSIGN_VSIX_PACKAGE=ON
# cmake --build .
# cmake --install .
