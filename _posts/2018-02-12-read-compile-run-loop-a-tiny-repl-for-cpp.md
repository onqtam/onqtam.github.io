---
title:  "Read-Compile-Run-Loop - a tiny REPL for C++"
date:   2018-02-12 16:01:34 +0200
header:
  overlay_image:  "/assets/images/rcrl_cover.jpg"
categories: programming
tags: [programming, hot reloading, cmake, C++, dll, repl]
excerpt: "How the cling alternative RCRL works and how to integrate it"
---

Ever wanted to modify some value or execute some (complex) statement while your C++ program is running just to test something out? Something that cannot be done through the debugger or wouldn't be trivial? Scripting languages have a REPL ([read-eval-print-loop](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop)) - for example frontend web developers use the "javascript console" of the browser to test things out which is available after pressing F12.

For C++ there is the [cling](https://github.com/root-project/cling) project developed by researchers at CERN but it is built on top of LLVM and to say that it isn't trivial to integrate in your application so everything is callable and to have that working on any platform and toolchain would be a huge understatement.

Out of frustration with the half-done bindings with the scripting language used at a past job I came up with an idea how to make something which behaves almost as an interactive C++ interpreter with very few restrictions - and the [RCRL](https://github.com/onqtam/rcrl) project is a demo application with GUI which demonstrates the technique. Without further ado here is a video showcasing it:

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
print(); // ======> will result in "4343"
```

In the scripting language world REPLs automatically try to print the "value" of the last statement. In RCRL we need to do the printing ourselves.

It should be noted that in the above example the variables may have been written also in *global* sections, but then if we submit more code for compilation as a separate step those globals would be initialized again - and that is what the vars section is for - persistent globals which are initialized only once. Also variable definitions in *once* sections aren't visible outside of them since they are just local variables in function scope. That will make more sense after the following section.

When we are done we can press the ```Cleanup``` button which will:
- call the destructors of all globals defined in ```vars``` sections in reverse order
- unload all the plugins (as explained later) in reverse order
- delete all the plugins from the filesystem

## How it works

Here is what happens each time you submit code:

1. A ```.cpp``` file is reconstructed with all previous *global* and *vars* sections (in the appropriate order) and then the newly submitted sections are also appended (including *once* sections). 
2. The source file is compiled as a plugin shared object (```.dll```) and links against the executable (more on that later)
3. The plugin is then copied with a different name pedending on which compilation this is (so we would end up with ```plugin_5.dll``` if we have previously submitted code for compilation 4 times)
4. The plugin is loaded by the host application and all globals defined in the ```.cpp``` file are initialized from top to bottom

The ```.cpp``` file always includes a header called ```rcrl_for_plugin.h``` located in ```<repo>/src/rcrl/``` which has a few macros and forward declarations to make everything work:

```c++
RCRL_SYMBOL_IMPORT void*& rcrl_get_persistence(const char* var_name);
RCRL_SYMBOL_IMPORT void   rcrl_add_deleter(void* address, void (*deleter)(void*));

// ...
```

Here is how code from the different sections ends up in the source file:

- *global* sections just go straight to the ```.cpp``` file being compiled
- *once* sections go in a lambda that is called while globals are initialized:

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

- *vars* sections - they are parsed so that for each variable definition the type/name/initializer are extracted. The code in the source file for ```int a = 5;``` looks like this:

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
    
    1. Here we first get a pointer to the persistent variable called ```a```.
    2. If that pointer is null (```a``` is defined in a newly submitted ```vars``` section) we allocate a new integer using the appropriate initializer and then add a deleter through a function pointer to be associated with ```a```.
    3. Then we return the pointer from the lambda and immediately dereference it to initialize a global reference with the name ```a```.
    
    That way we ensure that the global persistent variable will be initialized only once and its state will be preserved through the following compilations. In code below the ```vars``` section we can continue using the reference ```a``` as if it is a global variable ```a```.

## About the RCRL repository

The demo is tested on Windows/Linux/MacOS and uses OpenGL 2. To build it follow the instructions in the repository.

the demo is just an example that is meant to be used as a starting point - it doesn't try to be a one-size-fits-all solution

## How to integrate

the 5 defines from cmake

can even support sending code from elsewhere to the host app for compiling/loading

integrating this properly (and even more optimally) requires knowledge of build systems, compilers, static/dynamic libraries and more!

try to avoid linking the plugin to static libraries as build times will increase and there will be problems if there is global state in those libraries

requires c++14 in order to deduce the types of variables declared with c++11's auto keyword with the help of return type deduction
    can be built with VS 2015
    "rcrl requires c++11, but the whole demo requires c++14 because of third parties (ImGuiColorTextEdit)"

## Restrictions and notes

This is a list of unsupported C++ constructs/usages:

Lets see what C++ constructs/usages are unsupported.

In in ```vars``` sections:

- ```alignas()``` cannot be used
- C arrays can't be used - use ```std::array<>``` instead
- types should be allocatable - so no deleted operator ```new```/```delete``` for them
- multiple variables cannot be defined at once - like ```int a, b;```
- temporaries cannot bind to const references in the vars section - will get a compile error
- rvalue references as variables are disallowed


- pointers to objects from the RCRL - shouldn't be passed to the app because the cleanup will fuck shit up
    - use scope guards with code to be executed that removes the passed pointers to the host app
- don't rely on the addresses of functions - even if old dlls aren't unloaded and there isn't a crash - they won't be the same between iterations

- don't use the static keyword anywhere
- only variable definitions in the vars section
- raw string literals shouldn't be used in the vars section
- cannot assign lambdas to auto in the vars section - should use std::function<> for that
- decltype of variables from a vars section will return a reference to the type
- cannot handle vars with type auto* (explicit pointerness) - should be just auto

And here is a list of things to keep in mind:

- no global non-constexpr variables in the global section - variables only in the vars section! Otherwise they will always be re-initialized and their state wiped for the next code submission
- constexpr variables should be in ```global``` sections and not in ```vars``` because it wouldn't make any sense
- preprocessor use is allowed only in the ```once``` and ```global``` sections - and should be kept to a minimum

Some of these restrictions can be lifted by improving the parser.

## Room for improvement

- using libclang
- auto complete

- compiler error messages
- compiler warnings? o_O
- no debugging support...
- crash handling

## Random further thoughts

- This technique can be used for other compiled languages as well.

- It would be really cool to see something like this integrated into some big project like the [Unreal Engine](https://www.unrealengine.com) :)

- Wouldn't it be cool if applications had an optional module which enables interacting with them with C++? The module would be comprised of:
    - a custom version of the RCRL engine
    - a C++ compiler (perhaps the same version used for building them)
    - application API headers with exported symbols
    - application export lib so RCRL can link to it

- I'm betting that an awesome C++ game engine that enables an incredibly fast and flexible workflow can be developed without the need for scripting if 3 things are in place:
    - something like [dynamix](https://github.com/iboB/dynamix) is used for the object model which would allow for very flexible and elegant implementation of the business logic - one of the 2 main reasons people turn to scripting languages for that part
    - hot-reloading of any module (including whole subsystems like *rendering* and *physics*) and the ability to change the layout of types (like adding a new field in a class) at runtime is possible - perhaps with the help of a [practical generic reflection system](http://onqtam.com/programming/2017-09-02-simple-cpp-reflection-with-cmake/) - this is the second main reason people turn to scripting languages for parts of the business logic - reloadability
    - something like a REPL for convenience (RCRL fills this gap)
    
    What we lose by eliminating scripting is:
    - the ability to update clients remotely without modifying executables
    - game designers can no longer tinker with the logic in something easier than C++ - but in many studios this is the case anyway
    
    What we gain from eliminating scripting is:
    - the (imperfect) binding layer between C++ and the scripting is gone
    - no need for a virtual machine
    - optimal performance
    - programmers work in only 1 language
    - no code duplication (which is atleast in part inevitable otherwise)

Anyway - I'm eager to see what the C++ community thinks of this project/technique and what comes out of it!
