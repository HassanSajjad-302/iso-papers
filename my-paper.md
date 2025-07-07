---
title: "A New Approach For Compiling C++"
document: _______
date: 2023-09-22
audience: SG15
author:

  - name: HassanSajjad
    email: <hassan.sajjad069@gmail.com>
    toc: true
    toc-depth: 4

---

\pagebreak

# Abstract

This paper proposes an Inter-process communication mechanism between the compiler
and the build-system so that build-system can compile C++ source-files
without scanning them first.
This could result in improved compilation speed.

# Changes

## R1

Removed the older approach based on the compiler being a shared library and
added a new one based on inter-process communication.

## R0 (Initial)

# Introduction

For building C++20 modules and header-units,
build-systems rely on scanning the source-files and header-files first.
Scanning is needed to discover dependencies between the files.
So that build-system can establish the order between the compilations.
Otherwise, the user will have to edit these dependencies twice,
once in build-file and once in source-files.
This paper proposes a synchronous IPC mechanism to avoid the scanning.
Minimal changes are needed from the compiler to support this.

# Motivation and Scope

## Why is this faster?

While scanning a module-file, the compiler needs to maintain separate
macro contexts for every imported header-unit file.
This is different from a file including header-files where there is just one
macro context.
Due to this, scanning is slow for C++20 header-units.


# Impact On the Standard

This proposal does not impact the current build systems in any way.
Few build systems, however, might need non-trivial changes to support this.

# Design Decisions

## What are the tradeoffs?

1) Memory consumption of such an approach could be higher than the current approach.
   That is because more compilation processes need to be
   kept in the memory simultaneously.

However, this can be alleviated by the following:

In some build systems today, related files are grouped as one ```target```.
A ```target``` can depend on one or more other ```target```.
Such build systems can alleviate the higher memory consumption by ordering the
compilation of dependency target's files before the dependent target.
This way only the compilation process of dependency target's files
needs to be kept in the memory.
The compilation process of dependent target's files 
is initiated only after the compilation process of dependency target's files.

BMI files of a ```target``` are kept in the memory until there is no file of any dependent
target left to be compiled. At which point, this is cleared from the memory.

In case a similar file is being read by multiple compilations,
the memory consumption could be a little less than the current approach
as all such compilations can use one cached read instead of reading themselves.

2) In some cases, modifications to the configuration controlling the resource limitations of the build process might be
   needed as well.


3) Compiler can ask build-system to find header-include,
   which build-system can cache.
   Other compilations can benefit from this and not make filesystem calls themselves.

# Technical Specifications


```cpp

// CTB --> Compiler to Build-System
// BTC --> Build-System to Compiler

// string is 4 bytes that hold the size of the char array, followed by the array.
// vector is 4 bytes that hold the size of the array, followed by the array.
// All fields are sent in declaration order, even if meaningless.

// Compiler to Build System
// This is the first byte of the compiler to build-system message.
enum class CTB : uint8_t
{
    MODULE = 0,
    NON_MODULE = 1,
    LAST_MESSAGE = 2,
};

// This is sent when the compiler needs a module.
struct CTBModule
{
    string moduleName;
};

// This is sent when the compiler needs something else than a module.
// isHeaderUnit is set when the compiler knows that it is a header-unit.
// If findInclude flag is provided, then the compiler sends logicalName,
// Otherwise the compiler sends the full path.
struct CTBNonModule
{
    bool isHeaderUnit = false;
    string str;
};

// This is the last message sent by the compiler.
struct CTBLastMessage
{
    // Whether the compilation succeeded or failed.
    bool exitStatus = false;
    // Following fields are meaningless if the compilation failed.
    // header-includes discovered during compilation.
    vector<string> headerFiles;
    // compiler output
    string output;
    // compiler error output.
    // Any IPC related error output should be reported on stderr.
    string errorOutput;
    // exported module name if any.
    string logicalName;
    // This is communicated because the receiving process has no
    // way to learn the shared memory file size on both Windows
    // and Linux without a filesystem call.
    // Meaningless if the file compiled is not a module interface unit
    // or a header-unit.
    uint32_t fileSize = UINT32_MAX;
};

// Build System to Compiler
// Unlike CTB, this is not written as the first byte
// since the compiler knows what message it will receive.
enum class BTC : uint8_t
{
    MODULE = 0,
    NON_MODULE = 1,
    LAST_MESSAGE = 2,
};

struct BMIFile
{
    string filePath;
    uint32_t fileSize = UINT32_MAX;
};

struct ModuleDep
{
    BMIFile file;
    string logicalName;
};

// Reply for CTBModule
struct BTCModule
{
    BMIFile requested;
    vector<ModuleDep> deps;
};

struct HuDep
{
    BMIFile file;
    string logicalName;
    bool angled = false;
};

// Reply for CTBNonModule
struct BTCNonModule
{
    bool isHeaderUnit = false;
    string filePath;
    // if isHeaderUnit == false, the following three are meaning-less.
    bool angled = false;
    // if isHeaderUnit == true, fileSize of the requested file.
    uint32_t fileSize;
    vector<HuDep> deps;
};

// Reply for CTBLastMessage if the compilation succeeded.
struct BTCLastMessage
{
};
```

The following link has a sample implementation.
The implementation has 100% code coverage.
[https://github.com/HassanSajjad-302/ipc2978api](https://github.com/HassanSajjad-302/ipc2978api).
Below gives a summary of the design and some design rationale.

**TODO:**
API `IPCManagerBS` and `IPCManagerCompiler` should be documented
here as well.

On Windows, named pipes are a feature-rich and mature platform for IPC,
so it is used.
However, on Linux, Unix sockets are used.
As the pipe is unidirectional in Linux,
two pipes will be needed for bidirectional communication.
The name of the pipe is the object-file path except for header-units
for which it is the BMI path.
In the case of socket, the `sun_addr` of `sockaddr_un`
is `/tmp/` + hash of an object or bmi file-path that is converted
to 16 character hex-string.
Hash is calculated using [rapidhash](https://github.com/Nicoshev/rapidhash) library.
All of this is internal detail.
API on top of this is exposed through
`IPCManagerBS` and `IPCManagerCompiler`.
See `CompilerTest.cpp` and `BuildSystemTest.cpp` at the link given above
for the usage details. These tests are responsible for 100% code coverage.


The compiler is invoked with the compile-command 
to compile a module or header-unit file.
It should contain the option for the object file and the BMI file,
if any of this could be produced.
The compiler might not produce one or the other.
The compile-command does not contain any dependencies.
It contains a special flag that indicates the compiler to use this approach.
The name of such a flag could be ```noscanIPC```.

In this approach, the compiler can ask the build-system to find the
header-include on the filesystem.
Buildsystem can cache the results and share them with other compilations,
thus reducing the number of filesystem calls.
However, this feature is selective.
As it could be challenging for the build-system to
perfectly imitate the compiler in some scenarios.
So, this is to be controlled by a separate flag.
The name of such a flag could be ```findInclude```.

This approach also enables memory-mapped files / shared memory
for BMI files.
This reduces IO in cases where the BMI is read by multiple
compilations.
Only BMI is selected because source-file and header-files will
likely be read just once (to build header-unit or module), 
and these could potentially be edited while the build is
underway.
Similarly, object files are also read just once in most cases.

Now, when the compiler needs a module 
or needs to resolve a header-include,
 the compiler sends the respective message
and waits for the respective response.
If the file is BMI, then this is a shared memory object.
Besides the requested file,
the compiler also sends the dependencies of the file.
The `deps` only contains unique elements.
It does not contain an already sent dependency.
The compiler should not re-request an already sent file;
however, this is not an error.

Compiler sends ```CTBLastMessage``` once it has completed
the compilation or the compilation failed.
If the compilation succeeds
and compilation produces a BMI,
then compiler waits for the
```BTCLastMessage```, otherwise it can exit.
Compiler needs to wait for the ```BTCLastMessage```, so
that build-system can open the read-only shared memory object
before the compiler exits.

If the build-system does not receive ```CTBLastMessage```
before the pipe closes,
it concludes that an ICE happened.
In the case of ICE, the compiler should still try to send
`CTBLastMessage`.
The compiler should initialize the `IPCManagerCompiler`
right after detecting `noScanIPC` flag.
Because if the compiler crashes before the
`ConnectNamedPipe` or `accept` call completion on the build-system,
then that build-system thread might hang indefinitely.
Build-system may or may not detect this.