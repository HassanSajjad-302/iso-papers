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

This paper specifies the API for the interaction between the build system
and the compiler where the compiler is available as a shared library.
This could result in improved compilation speed.

# Changes

## R0 (Initial)

# Introduction

A very brief high level view of your proposal, understandable by C++ committee members who are not necessarily experts
in whatever domain you are addressing.

Today, almost all compilation happens by user or build system invoking the
compiler executable.
This paper proposes for the availability of the compiler as a shared library.
Both the compiler executable and the compiler shared library can co-exist.
The build tool can then interact with the shared library with the API specified
in this paper.
Compared to the current approach, this will result in faster compilation speed,
close to 25 - 40 % in some cases.

# Motivation and Scope

Why is this important? What kinds of problems does it address? What is the intended user community? What level of
programmers (novice, experienced, expert) is it intended to support? What existing practice is it based on? How
widespread is its use? How long has it been in use? Is there a reference implementation and test suite available for
inspection?

Most operating systems today support dynamic loading of the shared library.

## Why is this faster

i) API allows the build system to intercept the files read by the compiler.
So, the compilation of multiple files can use the same cached file.

ii) Module files need to be scanned to determine the dependencies of the file.
Only after that the files could be compiled in order.
In this approach, the scanning is not needed.

## How much faster this could be?

Tests conducted imitating real world scenarios revealed that it could be ```25 - 40 %``` faster
[https://github.com/HassanSajjad-302/solution5](https://github.com/HassanSajjad-302/solution5).

Tests were performed with C++20 MSVC compiler on Windows 11 operating system on modern
hardware at the time of writing.
Repositories used include SFML and LLVM.

The highlights include.

i) Estimated scanning time percent of the total compilation time
(scanning time + compilation time) per file for LLVM is ```8.46%```.

ii) SFML was compiled with C++20 header units.
Because compilation became faster, the scanning time took a larger proportion of the total time.
Scanning took ```25.72%``` of the total compilation time.
For few files scanning was actually slower than compilation.

iii) Estimated average scanning time per file for the project LLVM was ```217.9ms```.
For some files it was more than ```400ms```.

iv) On average the LLVM source file includes ```400``` header files.
```3598``` unique header files are read from the disk while compiling ```2854```
source files.
If LLVM is built with C++20 modules or C++20 header units,
there will be ```2854 + 3598``` process launches instead of ```2854```.
As more processes are needed for the compilation of module interface files or header units.
Few compilers use ```two phase``` model instead of ```one phase```.
Described
here [https://gitlab.kitware.com/cmake/cmake/-/issues/18355#note_1329192](https://gitlab.kitware.com/cmake/cmake/-/issues/18355#note_1329192).
In such a case, there will be ```2854 + (2 * 3598)``` process launches
instead of ```2854```.
By avoiding these costs of process setup and file reads should result in ```1 - 2 %```
compilation speed-up in a clean build in the project size of LLVM.

# Impact On the Standard

What other library components does does it depend on, and what depends on it? Is it a pure extension, or does it require
changes to standard components? Can it be implemented using current C++ compilers and libraries, or does it require
language or library features that are not part of C++ today?

This proposal does not impact the current build systems in any way.
Few build systems, however, might need non-trivial changes to support this.
Plans for supporting this in the build system HMake are explained here:
[https://lists.isocpp.org/sg15/att-2033/Analysis.pdf](https://lists.isocpp.org/sg15/att-2033/Analysis.pdf)

# Design Decisions

Why did you choose the specific design that you did? What alternatives did you consider, and what are the tradeoffs?
What are the consequences of your choice, for users and implementers? What decisions are left up to implementers? If
there are any similar libraries in use, how do their design decisions compare to yours?

## What are the tradeoffs?

Memory consumption of such an approach could be higher than the current approach.
That is because more ```compiler_state``` and the cached files need to be kept in the memory.

However, this can be alleviated by the following:

In some build systems today, related files are grouped together as ```target```.
A ```target``` can depend on one or more other ```target```.
Such build systems can alleviate the higher memory consumption by ordering the
compilation of files of the dependency before the dependent target.
This way only the ```compile_state``` of the files of the dependency target need to be kept
in the memory.
```compiler_state``` of the files of the dependent target only come in the picture once
the dependent target files are compiled.
ifcfile of a ```target``` are kept in the memory until there is no file of any dependent
target left to be compiled. At which point, this is cleared from the memory.

In case the similar file is being read by multiple compilations,
the memory consumption could be a little less than the current approach
as all such compilations can use one cached read instead of reading themselves.

# Technical Specifications

The committee needs technical specifications to be able to fully evaluate your proposal. Eventually these technical
specifications will have to be in the form of full text for the standard or technical report, often known as
“Standardese”, but for an initial proposal there are several possibilities:

Provide some limited technical documentation. This might be OK for a very simple proposal such as a single function, but
for anything beyond that the committee will likely ask for more detail.
Provide technical documentation that is complete enough to fully evaluate your proposal. This documentation can be in
the proposal itself or you can provide a link to documentation available on the web. If the committee likes your
proposal, they will ask for a revised proposal with formal standardese wording. The committee recognizes that writing
the formal ISO specification for a library component can be daunting and will make additional information and help
available to get you started.
Provide full “Standardese.” A standard is a contract between implementers and users, to make it possible for users to
write portable code with specified semantics. It says what implementers are permitted to do, what they are required to
do, and what users can and can’t count on. The “standardese” should match the general style of exposition of the
standard, and the specific rules set out in the Specification Style Guidelines, but it does not have to match the exact
margins or fonts or section numbering; those things will all be changed anyway if the proposal gets accepted into the
working draft for the next C++ standard.

Compilation pause and resume capability needs to be built into the compiler.

```cpp
namespace buildsystem
{

struct string
{
    const char *ptr;
    unsigned long size;
};

// Those char pointers that are pointing to the path are platform dependent i.e. whcar_t* in-case of Windows
struct compile_output
{
    // if (!completed), then pointer to the compiler state to be preserved by the build-system, else nullptr
    void *compiler_state;

    // if (completed), then compiler output and errorOutput, else nullptr
    string stdout;
    string stderr;

    // if (!completed), then the header-unit path the compiler is waiting on if any,
    // else if (completed && !error_occurred), then the pointer to the returned bmi_file if any, else
    // nullptr
    string header_unit_path_or_bmi_file;

    // if (!completed), then the name of the module the compiler is waiting on if any,
    // else if (completed && !error_occurred), then the pointer to the returned objectFile if any, else
    // nullptr
    string module_name_or_object_file;

    // if (completed && !error_occurred), then the logical_name of exported module if any.
    string logical_name;

    // Following is the array size, next is the array of header-includes.
    unsigned long header_includes_count;
    string *header_includes;

    // true if compilation completes, false otherwise
    bool completed;

    // if (completed), then true if an error occurred, false otherwise.
    bool error_occurred;
};

compile_output new_compile(string compile_command, string (*get_file_contents)(string file_path));
compile_output resume_compile(void *compiler_state, string bmi_file);
string get_object_file(string bmi_file);

} // namespace buildsystem
```

The compiler calls ```new_compile``` function passing it the
compile_command for the module file.
The compile command, however, does not include any dependencies.
If the compiler sees an import of a module, it sets the ```module_name_or_object_file``` string
of the ```compile_output``` return value.
If the compiler sees an import of a header unit, it sets the ```header_unit_path_or_bmi_file``` string
of the ```compile_output``` return value.
The build system now will preserve the ```compiler_state``` and
will check if the required file is already built, or it needs to be built, or it is being built.
Only after the file is available,
the build system will call ```resume_compile``` function passing it the BMI file.
```resume_compile``` is called until file has no dependency not provided and the compilation
succeeds.
If only the BMI file is returned and no object file on compilation completion,
the build system assumes that the compiler is using ```two phase``` model.
In this case, it will later call ```get_object_file``` to get the object file.
Distinction between the two models is discussed here:
https://gitlab.kitware.com/cmake/cmake/-/issues/18355#note_1329192.
The argument ```get_file_contents``` is used by the compiler to get file contents of any file
instead of reading itself.
This means that a file does not get read twice for different compilations.
As compilation completes, build-system will write ifc and object files to the disk as-well.

# Acknowledgements

# References

The template above is based on the one in N3370 Call for Library Proposals, which also has some other tips for writing a
good library proposal. Please note that the “Submission procedures” section in that document are outdated and should not
be used. The start of this page describes the new procedures.

\pagebreak

---
references:

- id: OpenPM
  citation-label: OpenPM
  title: "Open Pattern Matching for C++"
  author:
    - family: Solodkyy
      given: Yuriy
    - family: Reis
      given: [Gabriel, Dos]
    - family: Stroustrup
      given: Bjarne
      URL: http://www.stroustrup.com/OpenPatternMatching.pdf
- id: Mach7
  citation-label: Mach7
  title: "Mach7: Pattern Matching for C++"
  author:
    - family: Solodkyy
      given: Yuriy
    - family: Reis
      given: [Gabriel, Dos]
    - family: Stroustrup
      given: Bjarne
      URL: https://github.com/solodon4/Mach7
- id: PatMatPres
  citation-label: PatMatPres
  title: "\"Pattern Matching for C++\" presentation at Urbana-Champaign 2014"
  author:
    - family: Solodkyy
      given: Yuriy
    - family: Reis
      given: [Gabriel, Dos]
    - family: Stroustrup
      given: Bjarne
- id: SimpleMatch
  citation-label: SimpleMatch
  title: "Simple, Extensible C++ Pattern Matching Library"
  author:
    - family: Bandela
      given: John
      URL: https://github.com/jbandela/simple_match
- id: Patterns
  citation-label: Patterns
  title: "Pattern Matching in C++"
  author:
    - family: Park
      given: Michael
      URL: https://github.com/mpark/patterns
- id: Warnings
  citation-label: Warnings
  title: "Warnings for pattern matching"
  author:
    - family: Maranget
      given: Luc
      URL: http://moscova.inria.fr/~maranget/papers/warn/index.html
- id : YAMLParser
  citation-label : YAMLParser
  URL: http://llvm.org/doxygen/YAMLParser_8h_source.html
- id : SwiftPatterns
  citation-label : Swift Patterns
  title : "Swift Reference Manual - Patterns"
  URL: https://docs.swift.org/swift-book/ReferenceManual/Patterns.html

---
