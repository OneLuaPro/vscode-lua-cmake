# vscode-lua-cmake

This project provides a CMake-based automation layer for the complete build lifecycle of the [Lua Language Server VSCode extension](https://github.com/LuaLS/vscode-lua) for the Windows operating system. It handles everything from source acquisition to the final deployment-ready VSIX package. The primary goal of this project is to enable a seamless integration of [OneLuaPro](https://github.com/OneLuaPro) with VSCode by providing a tailored, predictable build of the language server as code-signed `vsix`- extension package.

## Features
- **Automated Workflow**: Clones, patches, and builds both server and client components.
- **Integrated Patching**: Automatically applies custom patches for client, server, and luamake.
- **Code Signing**: Optional support for Windows `signtool` and CodeSign tools for binaries and VSIX packages.
- **Superbuild Ready**: Designed to work standalone or as part of a larger CMake superbuild.
- **Clean Deployment**: Installs the final VSIX directly into the specified `CMAKE_INSTALL_PREFIX`.

## Differences from Original
This build environment introduces a few modifications to the default `vscode-lua` distribution:
- **Lua 5.5 by Default**: Configures the environment to use Lua 5.5 as the standard version.
- **Static Linking**: The server binary is built with statically linked runtime libraries to minimize external dependencies.
- **Appearance:** Replaces original icons and logos with custom **OneLuaPro** branding.
- **Publisher**: Set to *OneLuaPro* to distinguish it from the original *sumneko* publisher and prevent unintended automatic updates.

## License

See `https://github.com/OneLuaPro/vscode-lua-cmake/blob/master/LICENSE`.
