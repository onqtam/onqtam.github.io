---
title:  "Read-Compile-Run-Loop - a tiny REPL for C++"
date:   2018-02-12 18:01:34 +0200
header:
  overlay_image:  "/assets/images/rcrl_cover.jpg"
categories: programming
tags: [programming, hot reloading, cmake, C++, dll, repl]
excerpt: "How the cling alternative RCRL works and how to integrate it"
---

Ever wanted to modify some value or execute some (complex) statement while your C++ program is running just to test something out? Something that cannot be done through the debugger or wouldn't be trivial? Scripting languages have a REPL ([read-eval-print-loop](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop)) - for example frontend web developers use the "javascript console" of the browser to test things out which is available after pressing F12.

For C++ there is the [cling](https://github.com/root-project/cling) project developed by researchers at CERN but it is built on top of LLVM and to say that it isn't trivial to integrate in your application so everything is callable and to have that working on any platform and toolchain would be a huge understatement.

Out of frustration with the underdeveloped bindings with the scripting language used at a past job I came up with an idea how to make something which behaves almost as an interactive C++ interpreter with very few restrictions - and the [RCRL](https://github.com/onqtam/rcrl) project is a demo application with GUI which demonstrates the technique. Without further ado here is a showcasing video:

// insert youtube video here

First the contents of the precompiled header are shown and after that the exported API of the application.
Then we proceed with printing ```Hello world!``` with the output redirected to the right window, creating a new object and modifying it - all done at runtime. We change the current "section" with comments - that is explained below.

## How to use it

There are 3 types of sections of code that the user may submit for compilation:
- ```global``` - code that should be compiled in global scope goes there (like class/function definitions, includes, etc.)
- ```once``` - executable statements go in there - this is compiled in function scope and executed only once
- ```vars``` - definitions of variables that will be used later in other sections should go there

Sections are changed with single-line comments containing one of the 3 words ```global```/```once```/```vars```. For example:

```c++
// global
int foo() { return 42; }
// vars
int a = foo();
int& b = a;
// once
a++;
// global
#include <iostream>
void print() { std::cout << a << b << std::endl; }
// once
print(); // ======> will result in "4343" being printed
```

In the scripting language world REPLs automatically try to print the "value" of the last statement. In [RCRL](https://github.com/onqtam/rcrl) we need to do the printing ourselves.

It should be noted that in the above example the variables may have been written also in ```global``` sections, but then if we submit more code for compilation as a separate step, those globals would be initialized again and there would be multiple instances of them - and that is what ```vars``` sections are for - persistent globals which are initialized only once. Also variable definitions in ```once``` sections aren't visible outside of them since they are local variables in function scope. That will make more sense after the following section.

When we are done we can press the ```Cleanup``` button which will:
- call the destructors of all globals defined in ```vars``` sections in reverse order
- unload all the plugins (as explained later) in reverse order
- delete all the plugins from the filesystem

## How it works

Here is what happens each time you submit code:

1. A ```.cpp``` file is reconstructed with all previous ```global``` and ```vars``` sections (in the appropriate order) and then the newly submitted sections are also appended (including ```once``` sections) in their submission order. 
2. The ```.cpp``` file is compiled as a shared object (```.dll```) and links against the executable (more on that later)
3. The plugin is then copied with a different name depending on which compilation this is (so we would end up with ```plugin_5.dll``` if we have previously submitted code for compilation 4 times)
4. The plugin is loaded by the host application and all globals defined in the ```.cpp``` file are initialized from top to bottom

The ```.cpp``` file always includes a header called [```rcrl_for_plugin.h```](https://github.com/onqtam/rcrl/blob/master/src/rcrl/rcrl_for_plugin.h) located in [```<repo>/src/rcrl/```](https://github.com/onqtam/rcrl/tree/master/src/rcrl) which has a few macros and forward declarations to make everything work:

```c++
RCRL_SYMBOL_IMPORT void*& rcrl_get_persistence(const char* var_name);
RCRL_SYMBOL_IMPORT void   rcrl_add_deleter(void* address, void (*deleter)(void*));
// macros...
```

Here is how code from the different sections ends up in the source file:

- ```global``` sections just go straight to the ```.cpp``` file being compiled
- ```once``` sections go in a lambda that is called while globals are initialized:

    ```c++
RCRL_ONCE_BEGIN
a++;
RCRL_ONCE_END
    ```

    And after the preprocessor we get something like this:
    
    ```c++
int rcrl_anon_12 = []() {
a++;
return 0; }();
    ```

- ```vars``` sections - they are parsed so that for each variable definition the type/name/initializer are extracted. The source code for ```int a = 5;``` is this:

    ```c++
RCRL_VAR((int), (int), RCRL_EMPTY(), a, (5));
    ```
    
    which expands to the following after the preprocessor:

    ```c++
int& a = *[]() {
    auto& address = rcrl_get_persistence("a");
    if(address == nullptr) {
        address = (void*)new int(5);
        rcrl_add_deleter(address, [](void* ptr) { delete static_cast<int*>(ptr); });
    }
    return static_cast<int*>(address);
}();
    ```
    
    1. First we get a pointer (by ref) to the persistent variable with name ```a```.
    2. If that pointer is null (```a``` is defined in a newly submitted ```vars``` section) we allocate a new integer using the appropriate initializer and then add a deleter for ```a``` by passing a lambda as a function pointer.
    3. Then we return the pointer from the lambda and immediately dereference it to initialize a global reference with the name ```a```.
    
    That way we ensure that the global persistent variable will be initialized only once and its state will be preserved through the following compilations. In code below the ```vars``` section we can continue using the reference ```a``` as if it is a global variable ```a```.

The parser for ```vars``` sections is nothing special (it's not a [recursive descent parser](https://en.wikipedia.org/wiki/Recursive_descent_parser) and isn't written like usual parsers) but is just a few hundred lines of code and is surprisingly adequate - works with very complex types with lots of templates, ```decltype()```, ```auto```, references and complex initializers. I could have used something proper like [LibClang](https://clang.llvm.org/docs/Tooling.html#libclang) but decided to go with something custom and tiny instead.

## Restrictions

There are 2 sources of limitations for [RCRL](https://github.com/onqtam/rcrl):
- the parser for the ```vars``` sections is imperfect
- the method itself (shared objects, dynamic allocation of variables, etc.)

Let's see what C++ constructs/usages are unsupported. In ```vars``` sections:

- nothing else should go here except for variable definitions
- [*] ```alignas()``` cannot be used
- [*] C arrays can't be used - use ```std::array<>``` instead
- [*] multiple variables cannot be defined at once - like ```int a, b;```
- [*] don't use ```auto*``` - use ```auto``` directly and let it deduce pointer types
- [*] raw string literals shouldn't be used
- cannot assign lambdas to auto - should use ```std::function<>``` instead
- no deleted operator ```new```/```delete``` for types - they should be allocatable
- temporaries cannot bind to const references (and have their lifetime extended) because pointers are used under the hood - will get a compiler error when trying to get the address of a temporary
- rvalue references as variables are disallowed by the parser itself - related to the const reference restriction

The ```[*]``` entries can be removed by improving the parser or by using [LibClang](https://clang.llvm.org/docs/Tooling.html#libclang).

And here is a list of general restrictions to keep in mind:

- don't rely on the address of functions - it will be different after each recompilation and reloading
- don't pass pointers to functions or globals (and persistent globals from a ```vars``` section) to the host app without a way to remove them before doing a cleanup (or you would end up with dangling pointers)
- don't use the ```static``` keyword - it won't work as expected for local variables in function scope and it doesn't make sense for functions and globals
- ```decltype()``` of names from ```vars``` sections will return a reference to the type
- constexpr variables should be in ```global``` sections and not in ```vars``` because it wouldn't make any sense
- preprocessor use is allowed only in the ```once``` and ```global``` sections - and should be kept to a minimum
- global non-constexpr variables in the ```global``` section will be initialized (and have their state reset) each time code is submitted and a new plugin is compiled and loaded (also there will be multiple instances of them and the initializing code will be executed many times so if it has side effects they will accumulate) - use ```vars``` sections for proper persistence.
- class static variables should go into ```global``` sections outside of the class definition, but that means they will always be initialized - currently there isn't a way to make them persistent like globals from a ```vars``` section
- C++14 is required by the [RCRL](https://github.com/onqtam/rcrl) engine itself only for support of ```auto``` variables in ```vars``` sections (otherwise C++11 is enough)

Perhaps there are other issues I haven't found yet but all these seem minor.

## How to integrate

The [RCRL](https://github.com/onqtam/rcrl) repository is mainly a demo (tested on Windows/Linux/MacOS and uses OpenGL 2 - to build it follow the instructions in the repository). The important parts are the [RCRL](https://github.com/onqtam/rcrl) "engine" itself which is located in [```<repo>/src/rcrl/```](https://github.com/onqtam/rcrl/tree/master/src/rcrl) and it depends only on the [tiny-process-library](https://github.com/eidheim/tiny-process-library) third party which is a submodule of the repo and is located in [```<repo>/src/rcrl/third_party/```](https://github.com/onqtam/rcrl/tree/master/src/third_party). Everything else is for the demo project and the GUI. The plugin which is compiled by the engine also has a precompiled header ([```<repo>/src/precompiled_for_plugin.h```](https://github.com/onqtam/rcrl/blob/master/src/precompiled_for_plugin.h)) for speed of compilation.

The purpose of the whole repository is not to take the sources from [```<repo>/src/rcrl/```](https://github.com/onqtam/rcrl/tree/master/src/rcrl) as they are but to adapt them to your needs - the goal wasn't to create a one-size-fits-all solution because that is hardly possible.

The [RCRL](https://github.com/onqtam/rcrl) integration in the demo project is by no means optimal:
- it invokes CMake instead of the compiler directly
- the plugin is part of the whole CMake setup instead of being separated - and slow build systems like ```make``` and ```MSBuild``` take a lot of time to scan the dependencies (can be more than half a second) - unlike ```Ninja```

Integrating the [RCRL](https://github.com/onqtam/rcrl) engine properly and optimally requires knowledge of build systems, compilers, static/dynamic libraries and more!

Currently there are a few preprocessor identifiers setup from CMake for the [RCRL](https://github.com/onqtam/rcrl) engine to use for convenience (used in [```<repo>/src/rcrl/rcrl.cpp```](https://github.com/onqtam/rcrl/blob/master/src/rcrl/rcrl.cpp)):

- ```RCRL_PLUGIN_FILE``` - full path to the ```.cpp``` plugin source file for use by [RCRL](https://github.com/onqtam/rcrl)
- ```RCRL_PLUGIN_NAME``` - the name of the plugin target in CMake
- ```RCRL_BUILD_FOLDER``` - the root build directory of the whole CMake project
- ```RCRL_BIN_FOLDER``` - the folder where the plugin will be after compilation
- ```RCRL_EXTENSION``` - the platform-specific shared object extension (```.dll``` for Windows, ```.so``` for Linux, ```.dylib``` for MacOS)
- ```RCRL_CONFIG``` - only for multi-config IDEs like Visual Studio and XCode - the identifier represents the current configuration (Debug, Release, etc.)

The plugin needs to link to the executable so it can interact with the host application through the API with exported symbols. It also needs to link for the 2 functions exported by the [RCRL](https://github.com/onqtam/rcrl) engine for ```vars``` sections (```rcrl_get_persistence()``` and ```rcrl_add_deleter()```). In CMake executable targets cannot be linked against by default but this can be enabled by setting the ```ENABLE_EXPORTS``` target property to ```ON``` of the executable.

Actually the [RCRL](https://github.com/onqtam/rcrl) engine can reside also in some shared object and it is possible that the important parts of the host application API are implemented in other shared objects and not in the executable - in which case linking to it would be unnecessary (but still linking to the appropriate modules will be).

The entire "API" of [RCRL](https://github.com/onqtam/rcrl) is just a few functions in [```<repo>/src/rcrl/rcrl.h```](https://github.com/onqtam/rcrl/blob/master/src/rcrl/rcrl.h) which have a lot of comments for them and it is used in [```<repo>/src/main.cpp```](https://github.com/onqtam/rcrl/blob/master/src/main.cpp).

```c++
std::string cleanup_plugins(bool redirect_stdout = false);
bool submit_code(std::string code, Mode default_mode, bool* used_default_mode = 0);
bool is_compiling();
bool try_get_exit_status_from_compile(int& exitcode);
std::string get_new_compiler_output();
std::string copy_and_load_new_plugin(bool redirect_stdout = false);
```

Some ideas about the integration of the [RCRL](https://github.com/onqtam/rcrl) engine:
- the code editor may be separate from the host application - perhaps vim or whatever you'd like (so you can also have auto completion and whatever) - it doesn't have to be integrated into the host application
- the entire [RCRL](https://github.com/onqtam/rcrl) engine can be also outside of the host application - there should only be a way for the host application to be notified that a new plugin needs to be loaded (can be even done with a filesystem watcher)
- for performance: try to avoid linking the plugin to static libraries as build times will increase and global state in them might lead to problems
- for easy interaction with the host application most symbols should be exported - on Unix platforms all are exported by default from shared objects and linkable executables (unless built with ```-fvisibility=hidden```) but on Windows the opposite is default - so unless everything is annotated with [```__declspec(dllexport)```](https://msdn.microsoft.com/en-us/library/a90k134d.aspx) properly [this target property](https://cmake.org/cmake/help/latest/prop_tgt/WINDOWS_EXPORT_ALL_SYMBOLS.html) can be used in CMake to have everything exported by default (or [hacked](https://stackoverflow.com/questions/225432/export-all-symbols-when-creating-a-dll) for not CMake)

## Room for improvement

- ```global``` and ```vars``` sections can be merged if the parser is improved to be able to handle not just variable definitions (or perhaps it should be ditched entirely in favor of [LibClang](https://clang.llvm.org/docs/Tooling.html#libclang) - in which case we might even infer which pieces of the code are statements intended for function scope - to be executed only once - and ditch also the ```once``` sections)
- auto complete - but this is a big topic and is hard to make a universal solution that would fit everyone's needs
- crash handling - perhaps with structured exceptions under Windows when loading the plugin - in case anything happens while the code is executed
- compiler error messages - a mapping of lines between the plugin ```.cpp``` file and the submitted code can be made so errors can be highlighted in the original submitted source directly
- debugging support - with breakpoints and etc. - no idea about this...
- everything from the integration part of this post

## Random further thoughts

- This technique can be used for other compiled languages as well.

- It would be really cool to see something like this integrated into some big project like the [Unreal Engine](https://www.unrealengine.com) :)

- Wouldn't it be cool if applications had an optional module which enables interacting with them with C++? The module would be comprised of:
    - a custom version of the [RCRL](https://github.com/onqtam/rcrl) engine
    - a C++ compiler (perhaps the same version used for building them)
    - application API headers with exported symbols
    - application export lib so [RCRL](https://github.com/onqtam/rcrl) can link to it

- I'm betting that an awesome C++ game engine that enables an incredibly fast and flexible workflow can be developed without the need for scripting if 3 things are in place:
    - something like [dynamix](https://github.com/iboB/dynamix) is used for the object model which would allow for very flexible and elegant implementation of the business logic - one of the 2 main reasons people turn to scripting languages
    - hot-reloading of any module (including whole subsystems like *rendering* and *physics*) and the ability to change the layout of types (like adding a new field in a class) at runtime is possible - perhaps with the help of a [practical generic reflection system](http://onqtam.com/programming/2017-09-02-simple-cpp-reflection-with-cmake/) - this is the second main reason people turn to scripting languages for parts of the business logic - reloadability
    - something like a REPL for convenience ([RCRL](https://github.com/onqtam/rcrl) fills this gap)
    
    What we lose by eliminating scripting is:
    - the ability to update clients remotely without modifying executables
    - game designers can no longer tinker in something easier than C++ - but in many studios this is the case anyway (or perhaps they can - with something like [Unreal's Blueprints](https://docs.unrealengine.com/latest/INT/Engine/Blueprints/) - later compiled to C++)
    
    What we gain from eliminating scripting is:
    - the (imperfect) binding layer between C++ and the scripting is gone
    - no need for a virtual machine
    - optimal performance
    - programmers work in only 1 language
    - no code duplication (which is atleast in part inevitable otherwise)

Anyway - I'm eager to see what the C++ community thinks of this project/technique and what comes out of it!
