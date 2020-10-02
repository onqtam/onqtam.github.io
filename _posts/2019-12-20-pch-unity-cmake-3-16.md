---
title:  "CMake 3.16 added support for precompiled headers & unity builds - what you need to know"
date:   2019-12-20 15:40:34 +0200
header:
  overlay_image:  "/assets/images/cmake.png"
categories: programming
tags: [programming, C++, compile times, build systems, cmake]
excerpt: "A complete guide to applying the techniques + a few other tips"
---

Modules are coming in C++20 but it will take a while before they are widely adopted, optimized and supported by tooling - what can we do **right now**?

![](/assets/images/compiling.png)

I recently consulted a company on this exact matter - luckily [CMake 3.16 was just released](https://cmake.org/download/) and there was no need to resort to [3rd party CMake scripts](https://github.com/sakra/cotire) for precompiled headers and unity builds (thanks to [Cristian Adam](https://twitter.com/cristianadam) for the hard work - [MR 1](https://gitlab.kitware.com/cmake/cmake/merge_requests/3553), [MR 2](https://gitlab.kitware.com/cmake/cmake/merge_requests/3611)!!!). Here is what I told them:

# Precompiled headers (PCH)

### The idea is to *precompile* a bunch of common header files
- to *precompile* means that the compiler will parse the C++ headers and save its intermediate representation (IR) into a file, and then when compiling the ```.cpp``` files of the target that IR will be prepended to them - as if the headers were included - the contents of the PCH are the first thing each translation unit sees
- easy to integrate - doesn't require any C++ code changes
- ~20-30% speedup with GCC/Clang (can be up to 50%+ with MSVC)
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
- [```target_precompile_headers(<my_target> PRIVATE my_pch.h)```](https://cmake.org/cmake/help/latest/command/target_precompile_headers.html)
- the PCH will be included automatically in every ```.cpp``` file
    - adding a PCH to a target doesn't require that you remove the headers in it from all ```.cpp``` files - the C preprocessor is fast
- easiest if a single header includes the common ones - [example](https://github.com/onqtam/game/blob/master/src/precompiled.h)
    - you could have per-project precompiled header files or you could [reuse a PCH from one CMake target in another](https://cmake.org/cmake/help/latest/command/target_precompile_headers.html#reusing-precompile-headers) - remember that each PCH takes around ~50-200MB and takes some time to compile...
- you could list the headers which you want precompiled directly in the call to ```target_precompile_headers``` and even set them as ```PUBLIC```/```PRIVATE``` selectively so other targets which link to the current one would also precompile those, but I'm old fashioned and prefer to maintain the PCH for each target on my own.

### Some problems
- adding a header which was used only in 30% of the ```.cpp``` files to the precompiled header means that all ```.cpp``` files in the target will now have access to it - in time more files might start depending on it without you even noticing - the code might not build without the PCH anymore
    - do you care if it compiles successfully without a PCH? If so, make every ```.cpp``` explicitly include the precompiled header. This would be problematic if the same ```.cpp``` file is used in 2 or more CMake targets with different PCHs but in that case you should probably move that ```.cpp``` into a static lib, compile it only once and link against that!
- if you are using GCC but are using ```ccls```/```cquery```/```clangd```/```rtags``` or (based on clang) as a language server - those tools might not work because they will try to read the ```.gch``` file produced by GCC - [bug report](https://bugs.llvm.org/show_bug.cgi?id=41579)

# Unity builds

### The idea is to cram the ```.cpp``` files of a CMake target into a few ```.cpp``` files which include the original ```.cpp``` files
- up to 7-8 times faster builds (usually x3 or x4)
- example: a project of 200 ```.cpp``` files might be divided into 16 unity ```.cpp``` (or ```.cxx``` - whatever) files each including about 13 of the original ```.cpp``` files => 16 ```.cpp``` files to build in parallel and 16 ```.obj``` files to link
- the reasons for the speedup are:
    - common headers from the different ```.cpp``` files end up being included and parsed fewer times (this is beneficial even when using PCHs!)
    - common template instantiations with the same types in separate ```.cpp``` files (```vector<int>```) end up being done in fewer places
    - the linker has to stitch much fewer ```.obj``` files in the end - there are a lot less *weak* symbols to deduplicate (```inline```/template functions from headers end up in every ```.obj``` => linkers leave just 1)
        - surprisingly incremental builds (changing a single ```.cpp```) will probably also be faster instead of slower (even though you compile more ```.cpp``` files together) - precisely because of the reduced number of *weak* duplicated symbols!
    - less compiler invocations and less ```.obj``` files are written to disk
- the most reliable way to detect [ODR violations](https://stackoverflow.com/questions/31722473)
- runtime (final binary) might even be faster! (free [LTO](https://en.wikipedia.org/wiki/Interprocedural_optimization)) - because the compiler sees more symbols from different ```.cpp``` files together
- [A detailed blog post about unity builds and why they make builds faster](http://onqtam.com/programming/2018-07-07-unity-builds/)

### How to use
- CMake 3.16 adds the [```UNITY_BUILD```](https://cmake.org/cmake/help/latest/prop_tgt/UNITY_BUILD.html) target property
    - you can set this property per target explicitly
        - ```set_target_properties(<target> PROPERTIES UNITY_BUILD ON)```
    - or set it globally: ```set(CMAKE_UNITY_BUILD ON)``` (or call CMake with ```-DCMAKE_UNITY_BUILD=ON```) and then you can explicitly disable it for some targets by setting their property to ```OFF```
- the order in which the ```.cpp``` files go into the batches depends on the order they were given to a target in ```add_library```/```add_executable```
- if for some reason 2 ```.cpp``` files are hard to compile together they can be separated in different batches by reordering the sources
    - or use [```SKIP_UNITY_BUILD_INCLUSION```](https://cmake.org/cmake/help/latest/prop_sf/SKIP_UNITY_BUILD_INCLUSION.html) to exclude one
- about 10-20 ```.cpp``` files per unity is the most optimal
    - this is controlled through the [```UNITY_BUILD_BATCH_SIZE```](https://cmake.org/cmake/help/latest/prop_tgt/UNITY_BUILD_BATCH_SIZE.html) target property - default is 8 (can be set globally with [```CMAKE_UNITY_BUILD_BATCH_SIZE```](https://cmake.org/cmake/help/latest/variable/CMAKE_UNITY_BUILD_BATCH_SIZE.html))
    - don't worry if a target has few ```.cpp``` files - if it has more than 1 it would benefit from a unity build, + decent build systems like ninja will schedule ```.obj``` files from different targets to be built in parallel
- the unity ```.cpp``` files will go in the build directory - you don't have to maintain them or add them to version control

### Initial problems when trying to compile a project as unity
- some headers will be missing include guards or ```#pragma once```
- there will be static globals (or in anonymous namespaces) in different ```.cpp``` files with identical names which would clash
    - either rename them or put such globals into an additional namespace - perhaps with the name of the file: for ```GraphVisitor.cpp``` I would recommend ```GRAPH_VISITOR_CPP``` (putting static symbols inside of a named namespace in a ```.cpp``` keeps their linkage to *internal* , and the same is true with nesting anonymous namespaces into named ones)
- there will be symbol ambiguities
    - mostly because some ```.cpp``` file uses a namespace, and then some other ```.cpp``` file which ends up in the same unity ```.cpp``` cannot compile
    - either remove the ```using namespace ...``` stuff or fully qualify symbols where necessary (can use ```::``` to mean *from global scope*)
- some macros might have to be explicitly ```#undef```-ined at the end of the ```.cpp``` files where they are defined/used
    - take a look at [```UNITY_BUILD_CODE_BEFORE_INCLUDE```](https://cmake.org/cmake/help/latest/prop_tgt/UNITY_BUILD_CODE_BEFORE_INCLUDE.html) and [```UNITY_BUILD_CODE_AFTER_INCLUDE```](https://cmake.org/cmake/help/latest/prop_tgt/UNITY_BUILD_CODE_AFTER_INCLUDE.html)
    - or rewrite them to C++ constructs - macros are **$h1t** anyway
- another more obscure possible problem: if a static library gets built as a unity build => there are fewer ```.obj``` files stitched together, and if some other target links to the static library and uses only some symbols - from a few of the original ```.obj``` files, it might no longer link because now there are other symbols in the same ```.obj``` file where the needed symbols reside, and those other symbols might need some other symbols which are in some other static library
    - solution: find what else needs to be linked in
- unity build failures are rare after the initial cleanup (and if everyone uses unity builds locally there wouldn't be any)
- it took me about ~2 full days to compile about 2000 ```.cpp``` files in 20+ different CMake targets as unity in my current company ([NuoDB](https://www.nuodb.com/)])

### Recommended way to go about it
- I would recommend trying targets 1 by 1 by setting their target property ```UNITY_BUILD``` to ON - and not to enable it globally directly
- start with low batching - first try to get the project to compile with 4 ```.cpp``` files per unity, and then increase
- if you desire to have 20 ```.cpp``` files per batch in the end - go to atleast 40 or 50, clean the errors and then move back to 20
    - => future problems will be less likely - when a new ```.cpp``` file is added somewhere it changes which ```.cpp``` files get paired together
- unity builds should eventually become the default mode for building
    - there should be a separate CI build that checks that the project still compiles not as unity - checking for missing includes.

### Some problems
If you use ```-DCMAKE_EXPORT_COMPILE_COMMANDS=ON``` you get a file called ```compile_commands.json``` generated by CMake in the build folder
- that file contains compile commands for each translation unit - with all definitions and includes necessary for parsing the C++
- that file is used by tools such as ```ccls```/```cquery```/```clangd```/```rtags``` - language servers which are usually integrated with editors and IDEs such as ```vim```/```emacs```/```VSCode``` for intellisense (code completion, refactoring and syntax highlighting)
- when using unity builds the build information there will not contain compile commands for the specific ```.cpp``` files but only for the actual unity ```.cpp``` files - these tools might stop working correctly ([bug report](https://gitlab.kitware.com/cmake/cmake/issues/19826)). In my current company we have a wrapper script which does the calls to CMake like this:
    - calls CMake once with disabled unity builds + generation of the ```compile_commands.json``` file
    - calls CMake again with unity builds enabled + no ```compile_commands.json```
    - => the only downside of this is that if later we invoke our build system directly and it detects changes in CMake and reconfigures/regenerates the build files, it will be done with unity enabled & the generation of ```compile_commands.json``` disabled - perhaps leaving it stale

# Some other good tips to make builds faster

- use [ninja](https://ninja-build.org/) instead of GNU make - developed for building Google Chrome
    - superior scheduler, dependency tracking & change detection - optimal parallelization of building object files and linking targets - just a single standalone portable binary
    - CMake can generate ninja build files instead of Makefiles
        - just pass ```-G "Ninja"``` when calling CMake
        - to build the code either call ```cmake --build <path-to-build-dir>``` (instead of ```make -j```) or just ```ninja -C <path-to-build-dir>```
- change your compilers and linkers
    - experiment moving from gcc to clang
    - move from ```ld``` (the default linker) to ```gold``` (```-fuse-ld=gold```) or even ```lld``` (part of the LLVM project)
- use [dynamic linking](https://slides.com/onqtam/faster_builds#/62) instead of static - at least for internal builds if concerned about runtime performance
    - you could experiment with ```-fvisibility-inlines-hidden``` - [docs](https://gcc.gnu.org/wiki/Visibility)
- use [doctest](https://github.com/onqtam/doctest/) for unit tests ([benchmarks](https://github.com/onqtam/doctest/blob/master/doc/markdown/benchmarks.md)) - migrate from [googletest](https://github.com/onqtam/doctest/blob/master/doc/markdown/faq.md#how-is-doctest-different-from-google-test)/[Catch2](https://github.com/onqtam/doctest/blob/master/doc/markdown/faq.md#how-is-doctest-different-from-catch)
- [include-what-you-use](https://github.com/include-what-you-use/include-what-you-use)
    - based on clang - 100% correct parsing - helps identify unnecessary headers & where a simple forward declaration would do
- [Bloaty McBloatface](https://github.com/google/bloaty) - shows you a size profile of binaries
    - great to identify where code bloat is coming from and which symbols take the most space - which are the most offending templates?
- look into [```extern template``` from C++11](https://slides.com/onqtam/faster_builds#/41) ([blog](https://arne-mertz.de/2019/02/extern-template-reduce-compile-times/))
    - tells the compiler to not instantiate a specific template (for example ```std::vector<int>```) in the current translation unit
    - diagnosing which templates are a problem is easiest with:
        - [Bloaty McBloatface](https://github.com/google/bloaty), or any of [these](https://slides.com/onqtam/faster_builds#/45) [tools](https://slides.com/onqtam/faster_builds#/48)
- [caching](https://slides.com/onqtam/faster_builds#/67) & [distributed builds](https://slides.com/onqtam/faster_builds#/70)
- inspecting the physical structure of projects - targets & dependencies
    - [Graphviz](http://www.graphviz.org/) ([in CMake](https://cmake.org/cmake/help/latest/module/CMakeGraphVizOptions.html)) - ```cmake --graphviz=<file>```
    - [sourcetrail](https://www.sourcetrail.com/), or [other](https://slides.com/onqtam/faster_builds#/75) [tools](https://slides.com/onqtam/faster_builds#/22)
- PIMPL ([1](https://slides.com/onqtam/faster_builds#/27), [2](https://slides.com/onqtam/faster_builds#/28)), [disabling inlining for some functions](https://slides.com/onqtam/faster_builds#/38), rewriting templates... too much effort - do this as a last resort
- on the hardware side - more cores, more RAM... Duuh :D
    - use RAM disks (filesystem in your RAM) - every OS supports those. Put the compiler and the temp & output directories there

# Final thoughts

If it was up to me most of the techniques listed here would be put to use - from top to bottom - they are sorted based on impact and cost to implement. Slow builds don't just waste time - they also break the 'flow' (context switching) and discourage refactoring and experimentation - how do you put a price on that?

Based on my [```The Hitchhiker's Guide to Faster Builds```](https://www.youtube.com/watch?v=anbOy47fBYI) talk ([slides](https://slides.com/onqtam/faster_builds)).

You can checkout the reddit discussion for this article [here](https://www.reddit.com/r/cpp/comments/edaj7s/cmake_316_added_support_for_precompiled_headers/).

I'm available for hire to fix your C++ build times - you can expect something along the lines of whatever is in this blog post.
