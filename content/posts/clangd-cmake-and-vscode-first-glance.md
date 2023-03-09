---
title: "Clangd, Cmake and Vscode first glance - solved some configuration errors"
date: 2023-03-09T11:50:22+08:00
draft: false
tags:
    - se
    - cmake
    - clangd
---

## cmake: how to add a pure header lib

INTERFACE is what we need:

```cmake
add_library(<target> INTERFACE)
target_include_directory(<target> 
    INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    INTERFACE $<INSTALL_INTERFACE:include>
    )
```

## clangd

### gcc and g++ version not match

clangd will find g++ version based on latest gcc version.
In my case, gcc-12 is installed becaure the WIFI adapter driver depends it, but not g++-12.

The solution is to install g++-12 or uninstall gcc-12.

### clangd cannot find headers of a cmake project

add a line in `<project_root>/CMakeLists.txt`:

```cmake
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
````

Re-run cmake, this generates a file `compile_commands.json` in the build directory.

In the source directory, create a symbolic link to it:

```sh
ln -s build/compile_commands.json compile_commands.json
```

Reload the IDE.
