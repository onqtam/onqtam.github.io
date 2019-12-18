---
title:  "CMake 3.16 added support for precompiled headers & unity builds - what you need to know"
date:   2019-12-12 16:40:34 +0200
header:
  overlay_image:  "/assets/images/cmake.png"
categories: programming
tags: [programming, C++, compile times, build systems, cmake]
excerpt: "A weird tree-like guide to applying the techniques"
---

Modules are coming in C++20 but it will take a while before they are widely adopted, optimized and supported by tooling - what can we do right now to speed up our builds?

I recently consulted a company on this exact matter - luckily CMake 3.16 was just released and there was no need to resort to 3rd party CMake scripts from GitHub for precompiled headers and unity builds (such as [cotire](https://github.com/sakra/cotire) - more than 4k LOC of CMake...). Here is what I told them about these 2 techniques in a weirdly-tree-like-structured way:

## Precompiled headers (PCH)

### The idea is to "precompile" a bunch of common header files
- "precompile" means that the compiler will parse the C++ headers and save its intermediate representation (IR) into a file
    - then when compiling sources that IR will be prepended to them - as if the headers were included
        - the PCH is the first thing each translation unit sees
- easy to integrate - doesn't require any C++ code changes
- ~20-30% speedup on UNIX (can be up to 50%+ with MSVC)
- for targets with at least 10 ```.cpp``` files (takes space & time to compile)
### What to put in a PCH
- STL & third-party libs like boost (used in at least ~30% of the sources)
- some **rarely changing** project-specific headers (at least 30% use)
    - for example if you have common utilities for logging/etc.
- each time any header which ends up in the PCH is changed - the entire PCH is recompiled along with the entire target which includes it
- careful not to put too much into a PCH - once it reaches ~150-200MB you might start hitting diminishing returns
- how to determine which are the most commonly used header files
    - option 1: do a few searches in the codebase/target
        - ```<algorithm>```, ```<vector>```, ```<boost/asio.hpp>```, etc.
        - note that some header might be included only in a few other header files, but if those headers go everywhere, then the other header gets included almost everywhere as well
    - option 2: - [use software to visualize includes & dependencies](https://slides.com/onqtam/faster_builds#/22)
### How to use
- [```target_precompile_headers(<my_target> PRIVATE my_pch.h)```](https://cmake.org/cmake/help/v3.16/command/target_precompile_headers.html)
- the PCH will be included automatically in every ```.cpp``` file
    - adding a PCH to a target doesn't require that you remove the headers in it from all ```.cpp``` files - the C preprocessor is fast
- easiest if a single header which includes the common ones - [example](https://github.com/onqtam/game/blob/master/src/precompiled.h)
    - you could have per-project precompiled header files
        - or you could [reuse a PCH from one CMake target in another](https://cmake.org/cmake/help/v3.16/command/target_precompile_headers.html#reusing-precompile-headers)
            - remember that each PCH takes around ~50-200MB and takes some time to compile... reuse is good!
- you could list the headers which you want precompiled directly in the call to ```target_precompile_headers``` and even set them as ```PUBLIC```/```PRIVATE``` selectively so other targets which link to the current one would also precompile those, but I'm old fashioned and prefer to maintain the PCH for each target on my own.
### Some random final notes:
- adding a header which was used only in 30% of the ```.cpp``` files to the precompiled header means all ```.cpp``` files in the taret will have access to it - in time more files might start depending on it without you even noticing - the code might not build without the PCH anymore
    - do you care if it compiles successfully without a PCH?
        - make every ```.cpp``` explicitly include the precompiled header
            - problematic if the same ```.cpp``` file is used in 2 or more CMake targets with different PCHs
                - move that ```.cpp``` into a static lib, compile it only once and link against that!
- if you are using GCC but are using ccls/clangd or anything else as a language server which is based on clang:
    - those tools might not work because they will try to read the .gch file produced by GCC - [bug report](https://bugs.llvm.org/show_bug.cgi?id=41579) (along with a patch)

## Unity builds

- [detailed blog post about unity builds and why they make builds faster](http://onqtam.com/programming/2018-07-07-unity-builds/)
- the idea is to cram the ```.cpp``` files of a CMake target into a few ```.cpp``` files which include the original ```.cpp``` files
    - up to 7-8 times faster builds (usually x3 or x4)
    - the reasons for the speedup are:
        - common headers from the different ```.cpp``` files end up being included and parsed fewer times
            - this is beneficial even when using precompiled headers
        - common template instantiations with the same types in separate ```.cpp``` files (```vector<int>```) end up being done in fewer places
        - the linker has to stitch much fewer ```.obj``` files in the end
            - also there are a lot less "weak" symbols to remove
                - "inline"/template functions from headers end up in every ```.obj``` => linkers remove all duplicates and leave just 1
        - less compiler invocations and less ```.obj``` files are written to disk
            - even incremental builds (changing a single .cpp) will probably be faster (even though you compile more ```.cpp``` files together)
    - example: a project of 200 ```.cpp``` files might be divided into 16 unity ```.cpp``` files each including about 13 ```.cpp``` files
        - => 16 ```.cpp``` files to build in parallel and 16 ```.obj``` files to link
    - [uncovers ODR violations](https://stackoverflow.com/questions/31722473)
    - runtime (final binary) might be faster (free LTO!) - because the compiler sees more symbols from different ```.cpp``` files together
- how to use
    - CMake 3.16 adds the ```UNITY_BUILD``` target property
        - make sure to read this page: https://cmake.org/cmake/help/latest/prop_tgt/UNITY_BUILD.html
        - you can set this property per target explicitly
            - set_target_properties(<target> PROPERTIES UNITY_BUILD ON)
            - or set it globally: "set(CMAKE_UNITY_BUILD ON)" (or call CMake with "-DCMAKE_UNITY_BUILD=ON")
                - and then you can explicitly disable it for just a few targets by setting their property
    - the order in which the ```.cpp``` files go into the batches depends on the order they were given to a target in add_library/add_executable
    - if for some reason 2 ```.cpp``` files are hard to compile together they can be separated in different batches
        - or one of them can be excluded by setting the SKIP_UNITY_BUILD_INCLUSION property on it
            - https://cmake.org/cmake/help/latest/prop_sf/SKIP_UNITY_BUILD_INCLUSION.html
    - about 10-20 ```.cpp``` files per unity is the most optimal
        - this is controlled through the UNITY_BUILD_BATCH_SIZE target property - default is 8
            - https://cmake.org/cmake/help/v3.16/prop_tgt/UNITY_BUILD_BATCH_SIZE.html
            - can be set globally with CMAKE_UNITY_BUILD_BATCH_SIZE
        - don't worry if a target has few ```.cpp``` files - if it has >1 ```.cpp``` file it would benefit from a unity build
            - also a good build system such as ninja will schedule ```.obj``` files from different targets to be compiled in parallel
    - the unity .cxx files will go in the build directory - you don't have to maintain them or add them to source control
- initial problems when trying to compile a project as unity
    - some headers will be missing include guards or "#pragma once"
    - there will be static globals (or in anonymous namespaces) in different ```.cpp``` files with identical names - those would clash
        - either rename them or put such globals into an additional namespace - perhaps with the name of the file: GRAPH_VISITOR_CPP
            - putting static symbols inside of a named namespace keeps their linkage to "internal" (same with nesting anonymous namespaces into named ones)
    - there will be symbol ambiguities
        - mostly because some ```.cpp``` file uses a namespace, and then some other ```.cpp``` file which ends up in the same unity ```.cpp``` cannot compile
        - either remove the "using namespace ..." stuff or fully qualify symbols where necessary (can use "::" to mean "from global scope")
    - some macros might have to be explicitly #undef-ed at the end of the ```.cpp``` files where they are defined/used
        - or rewritten to C++ constructs - macros are $h1t anyway
    - another more obscure possible problem:
        - if a static library gets built as a unity build - there are fewer ```.obj``` files stitched together
            - if some other target links to the static library and uses only some symbols - from a few of the original ```.obj``` files
                - it might no longer link because now there are other symbols in the same ```.obj``` file where the needed symbols reside
                    - those other symbols might need some other symbols which are in some other static library
                    - solution: find what else needs to be linked against - because of this newly dragged dependency
    - unity build failures are rare after the initial cleanup (and if everyone uses unity builds locally there wouldn't be any)
    - it took me about 3-4 full days to compile about 2000 ```.cpp``` files in 20 different CMake targets to unity in my current company
        - there was plenty of 10+ year old code
- recommended way to go about it
    - I would recommend trying targets 1 by 1 by setting their target property UNITY_BUILD to ON - and not to enable it globally directly
    - start with low batching - get the project to compile with 4 ```.cpp``` files per unity, then increase
    - if you desire to have 20 ```.cpp``` files per batch in the end - go to atleast 40-50, clean the errors and then move back to 20
        - ==> future problems will be less likely - when a new ```.cpp``` file is added somewhere and changes which ```.cpp``` files get paired together
    - unity builds should eventually become the default mode for building for all developers
        - there should be a separate CI build that checks that the project still compiles not as unity - checks for missing includes.
- if you are using CMAKE_EXPORT_COMPILE_COMMANDS=ON you get a file called "compile_commands.json" generated by CMake in the build folder
    - that file contains compile commands for each translation unit - with all definitions and includes
    - that file is used by tools such as ccls/cquery/clangd - language servers which are usually integrated with editors and IDEs such as vim/emacs/VSCode for intellisense (code completion, refactoring and syntax highlighting)
        - when using unity builds the build information there will not contain compile commands for the specific ```.cpp``` files but only for the actual unity .cxx files - tools such as ccls and cquery might stop working correctly
            - in my current company we have a script which calls cmake - this is what we do:
                - call CMake once with disabled unity builds + to generate the compile_commands.json file
                - call CMake again with unity builds enabled + no generation for that file
    - this will probably get fixed eventually: https://gitlab.kitware.com/cmake/cmake/issues/19826

## Some other good tips to make builds faster

- use ninja instead of GNU make
    - originally developed in Google for building Google Chrome - much better than make
    - superior scheduler, dependency tracking & change detection - optimal parallelization of building object files and linking targets
    - CMake can generate ninja build files instead of Makefiles
        - just pass -G "Ninja" when calling CMake
        - ninja is a standalone portable binary
            - should be available as a package "ninja" or "ninja-build" in linux distros
            - or download it from the latest release: https://github.com/ninja-build/ninja/releases
        - to build the code either call "cmake --build <path-to-build-dir>" (instead of "make -j") or just "ninja -C <path-to-build-dir>"
- change your compilers and linkers
    - experiment moving from gcc to clang
    - check if you are using "ld" - the default linker, and if so, switch to "gold" or even "lld" (part of the LLVM project)
        - "-fuse-ld=gold" if gold is installed on the system
- use dynamic linking instead of static - at least for internal builds if concerned about runtime performance
    - https://slides.com/onqtam/faster_builds#/62
    - you could experiment adding "-fvisibility-inlines-hidden" to the compiler flags
        - documentation: https://gcc.gnu.org/wiki/Visibility
- use doctest for unit tests - https://github.com/onqtam/doctest/ - [benchmarks](https://github.com/onqtam/doctest/blob/master/doc/markdown/benchmarks.md)
    - migrating from googletest: https://github.com/onqtam/doctest/blob/master/doc/markdown/faq.md#how-is-doctest-different-from-google-test
    - migrating from Catch2: https://github.com/onqtam/doctest/blob/master/doc/markdown/faq.md#how-is-doctest-different-from-catch
- include-what-you-use - https://github.com/include-what-you-use/include-what-you-use
    - based on clang - parses the source code 100% correctly and helps identify unnecessary headers and helps identify where a simple forward declaration would do
- bloaty - https://github.com/google/bloaty
    - Bloaty McBloatface will show you a size profile of the binary so you can understand what's taking up space inside.
    - great to identify where code bloat is coming from and which symbols take the most space - which are the offending templates?
- look into "extern template" from C++11
    - tells the compiler to not instantiate a specific template (for example ```std::vector<int>```) in the current translation unit
    - https://slides.com/onqtam/faster_builds#/41
    - https://arne-mertz.de/2019/02/extern-template-reduce-compile-times/
    - diagnosing which templates are a problem is easiest with:
        - bloaty
        - https://slides.com/onqtam/faster_builds#/45
        - https://slides.com/onqtam/faster_builds#/48
- caching & distributed builds
    - https://slides.com/onqtam/faster_builds#/67 
    - https://slides.com/onqtam/faster_builds#/70   
- inspecting the physical structure of projects - targets & dependencies
    - https://www.sourcetrail.com/
    - "cmake --graphviz=<file>"
        - https://cmake.org/cmake/help/latest/module/CMakeGraphVizOptions.html
        - http://www.graphviz.org/
    - https://slides.com/onqtam/faster_builds#/75
    - https://slides.com/onqtam/faster_builds#/22
- PIMPL, disabling inlining for some functions, rewriting templates... - too much effort and little gain - do this as a last resort.
- on the hardware side
    - more cores, more RAM...
    - use RAM disks (filesystem in your RAM) for builds - every OS supports those. Put the compiler and the temp & output directories there.

## Final thoughts

If it was up to me most of the techniques listed here would be put to use - from top to bottom - they are sorted based on impact and cost to implement. Slow builds don't just waste time - they also break the 'flow' (context switching) and discourage refactoring and experimentation - how do you put a price on that?

Most of the things here are based on my "The Hitchhiker's Guide to Faster Builds" talk:
- slides: [https://slides.com/onqtam/faster_builds](https://slides.com/onqtam/faster_builds)
- recording: [https://www.youtube.com/watch?v=anbOy47fBYI](https://www.youtube.com/watch?v=anbOy47fBYI)


