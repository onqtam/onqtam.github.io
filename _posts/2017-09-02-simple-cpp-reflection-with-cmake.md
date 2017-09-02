---
title:  "Simple C++ reflection with CMake"
date:   2017-09-02 21:04:34 +0200
header:
  overlay_image:  "/assets/images/reflection.png"
categories: programming
tags: [programming, reflection, cmake]
excerpt: "Adding reflection to codebases without heavy template metaprogramming or external tools such as LibClang"
---

Reflection is a very useful tool and if you aren't familiar with the state of affairs in C++ I'd suggest reading [these](http://jackieokay.com/2017/04/13/reflection1.html){:target="_blank"} [two](http://jackieokay.com/2017/05/06/reflection2.html){:target="_blank"} excellent blog posts by [@jackie](https://twitter.com/jackayline){:target="_blank"}.

Unfortunately none of the current solutions really clicks with me:
- using [Boost.Fusion](http://www.boost.org/doc/libs/1_65_0/libs/fusion/doc/html/fusion/adapted/define_struct.html){:target="_blank"} or [Boost.Hana](http://www.boost.org/doc/libs/1_65_0/libs/hana/doc/html/index.html#tutorial-introspection-adapting){:target="_blank"} to annotate my classes:
    - seems a bit ugly and intrusive
    - raises concerns about compile times
    - no way to annotate fields in a custom way
- external tooling (such as libClang - see [siplasplas](https://github.com/Manu343726/siplasplas){:target="_blank"} and [CPP-Reflection](https://github.com/AustinBrunkhorst/CPP-Reflection){:target="_blank"}):
    - makes build setups complicated
    - not sure about the impact on compile times
    - no way to annotate fields in a custom way (unless I modify Clang to understand ```[[custom_tag]]```)

I also tried with a DSL (not in C++) which got parsed by a python script and C++ code got emitted but found it too much work to make it support complex C++ types - teaching it about templates, variants and typedefs just got hairy very quickly (not a parser expert).

## What I came up with

The [cmake-reflection-template](https://github.com/onqtam/cmake-reflection-template){:target="_blank"} repository is a small working example of a few source files with added reflection which generates serialization and deserialization routines (using ```std::any<>``` - so it requires C++17 - but it can be rewritten to serialize to JSON instead).

- each CMake target that wants to have reflection should have the [```target_parse_sources()```](https://github.com/onqtam/cmake-reflection-template/blob/master/scripts/utils.cmake#L16){:target="_blank"} CMake function called on it [like so](https://github.com/onqtam/cmake-reflection-template/blob/master/CMakeLists.txt#L10){:target="_blank"}
- each source file in the reflected projects has an attached custom CMake command so when it gets modified that command gets ran
- that command runs the parser on the file - named for example ```my_type.h``` - which generates code and dumps it in a file called ```my_type.h.inl``` in a ```gen``` folder inside [```CMAKE_BINARY_DIR```](https://cmake.org/cmake/help/latest/variable/CMAKE_BINARY_DIR.html){:target="_blank"}
- the resulting ```my_type.h.inl``` can be included either directly in ```my_type.h``` or perhaps elsewhere - the forward declarations of the generated functions are written inside the classes in ```my_type.h``` using the helper ```FRIENDS_OF_TYPE(MY_TYPE);``` macro.

It is a bit like what [Unreal is doing for reflection of properties](https://www.unrealengine.com/en-US/blog/unreal-property-system-reflection){:target="_blank"} - C++ source code is parsed (and annotated with preprocessor identifiers) and each source file includes the generated code for itself.

Here is what C++ code looks like with my solution:

```c++
class Foo {
    FRIENDS_OF_TYPE(Foo); // friend declarations of the generated functions

    FIELD int                a;
    FIELD float              b = 42.f;
    FIELD std::string        c {""};
    FIELD std::map<int, int> field_with_spaces_in_its_type;
    FIELD int
                             field_on_the_next_line_of_its_type;
};

#include <gen/my_type.h.inl>
```

```FIELD``` is a preprocessor identifier that expands to nothing - used for easier parsing - otherwise detecting comments or distinguishing between class fields and local variables in inline methods would be very complicated.

What gets generated for this class entirely depends on the parser. ```FIELD```, ```FRIENDS_OF_TYPE()``` and a few other preprocessor identifiers have to be visible in any source file that wants to use reflection - see them [here](https://github.com/onqtam/cmake-reflection-template/blob/master/src/common.h#L10-L19){:target="_blank"}.

Currently the parser from the example project would generate the following code for the type ```Foo``` written above:

```c++
any serialize(const Foo& src) {
    map<string, any> out;
    out["a"] = serialize(src.a);
    out["b"] = serialize(src.b);
    out["c"] = serialize(src.c);
    out["field_with_spaces_in_its_type"] = serialize(src.field_with_spaces_in_its_type);
    out["field_on_the_next_line_of_its_type"] = serialize(src.field_on_the_next_line_of_its_type);
    return out;
}
void deserialize(const any& src, Foo& dst) {
    const auto& data = any_cast<const map<string, any>&>(src);
    if(data.count("a")) { deserialize(data.at("a"), dst.a); }
    if(data.count("b")) { deserialize(data.at("b"), dst.b); }
    if(data.count("c")) { deserialize(data.at("c"), dst.c); }
    if(data.count("field_with_spaces_in_its_type")) { deserialize(data.at("field_with_spaces_in_its_type"), dst.field_with_spaces_in_its_type); }
    if(data.count("field_on_the_next_line_of_its_type")) { deserialize(data.at("field_on_the_next_line_of_its_type"), dst.field_on_the_next_line_of_its_type); }
}
```

[cmake-reflection-template](https://github.com/onqtam/cmake-reflection-template){:target="_blank"} is meant to be used as an initial starting point and it is expected to be modified by anyone using it to better suit their needs - perhaps to tweak the parsing, to use JSON instead of ```std::any<>```, to add prefixes to the macros or to change what code gets emitted. The CMake part is solid and the rest can be viewed as a proof of concept. One might even rewrite the parser in a different language! I'm no expert in templating engines or writing parsers so the python script (located in [```/scripts/```](https://github.com/onqtam/cmake-reflection-template/tree/master/scripts){:target="_blank"}) is nothing special but what I've got is good enough for my needs.

## Support for attributes and tags

Currently besides ```FIELD``` there are a few other preprocessor identifiers in a common header that expand to nothing - used for annotating:

```c++
#define FIELD    // indicates the start of a field definition inside of a type
#define INLINE   // class attribute - emitted functions should be marked as inline
#define CALLBACK // field attribute - call the callback after the field changes
#define ATTRIBUTES(...) // comma-separated list of attributes and tags into this
```

And they are used like this:

```c++
ATTRIBUTES(INLINE)
class Foo {
    FRIENDS_OF_TYPE(Foo);
    
    ATTRIBUTES(tag::special, CALLBACK(Foo::some_callback))
    FIELD std::string tagged;
    FIELD std::string normal;
    
    static void some_callback(Foo& src) { cout << "callback called!" << endl; }
};
```

This will result in the following codegen:

```c++
inline void print(const Foo& in) {
    print(in.tagged, tag::special()); // tag given to the print() function
    print(in.normal);
}
inline void deserialize(const any& src, Foo& dst) {
    const auto& data = any_cast<const map<string, any>&>(src);
    if(data.count("tagged")) { deserialize(data.at("tagged"), dst.tagged); Foo::some_callback(dst); }
    if(data.count("normal")) { deserialize(data.at("normal"), dst.normal); }
}
```

- The 2 routines are inline because of the class attribute - the codegen can be included in a header, which in turn can be included in many places.
- Both fields are of type ```std::string``` but [tagging](https://github.com/onqtam/cmake-reflection-template/blob/master/src/common.h#L22){:target="_blank"} would allow for different overloads to be chosen - for example consider there are 2 string fields which represent paths for assets of different types (meshes and textures) - if the reflection generates GUI for editing those fields one might want to have different filters for the 2 kinds of assets (```.jpg``` vs ```.mesh```)
- There is a callback attached to the ```tagged``` field and it will be called any time that field is changed in a routine generated by the reflection - this might be useful when 2 fields are related and changing one must result in changes to the other (the other might not even be annotated with ```FIELD```).

## Dependencies and minimal rebuild

The CMake macro ```target_parse_sources()``` adds custom CMake commands on each source file and when a source file is modified - the parser is ran only on that file - that way there is no explosion in build times. Also the parser is very light and its result will most likely be included by the changed file anyway so rebuilds will stay minimal.

Also when the parser itself is modified - all generated code in ```${CMAKE_BINARY_DIR}/gen``` is deleted since the parsing rules and codegen are most likely changed.

## Current limitations

- Types in namespaces not supported (but should be easy to extend).
- Nested types not supported (maybe easy).
- ```FIELD``` has to be used when declaring class fields - otherwise they will be skipped. This greatly simplifies the parser - but I'd be happy if the parser improved and all fields got parsed by default - and perhaps skipping fields should happen only when annotated explicitly to be skipped.
- Currently all generated ```.inl``` files go into the ```gen``` folder of the CMake build folder - that means that if 2 files in any of the projects (CMake targets) are named identically - there would be a clash. This can be alleviated if the CMake script is improved a bit.
- Cannot declare multiple fields on the same line: ```FIELD int x, y, z;``` - but I don't like that style anyway.
- Parser is not multi-line comment aware - will parse types in ```/*...*/``` and emit code for them

## How I use this

In my (cleverly titled) [game](https://github.com/onqtam/game){:target="_blank"} project I use reflection extensively. So far I generate only serialization/deserialization and GUI binding routines (3 functions per type) on top of which I've built the following:

- reloadable .dll plugins (in conjunction with the use of [dynamix](https://github.com/iboB/dynamix){:target="_blank"}) - even with the ability to change the layout of types at runtime! (by serializing object state, recreating them and then deserializing the old state)
- generic undo/redo system

I also like how composable everything is with the serialization/deserialization routines. In a few headers I provide overloads for the primitive types and for a few containers like ```std::vector<>``` (and maybe types from third parties) - and from then on the generated routines by the reflection for all my types call overloads for each of the fields of my types - composes quite nicely - I originally heard of such composing in [this talk](https://www.youtube.com/watch?v=Njjp_MJsgt8){:target="_blank"} about [```hash_append```](https://github.com/HowardHinnant/hash_append){:target="_blank"}!

I might expand on that in a future post.

Thanks for reading :)
