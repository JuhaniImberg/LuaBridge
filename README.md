# LuaBridge

[LuaBridge][3] is a lightweight, dependency-free library for making C++ data,
functions, and classes available to Lua. It works with Lua revisions starting
from 5.1.2. [Lua][5] is a powerful, fast, lightweight, embeddable scripting
language.

LuaBridge offers the following features:

- Nothing to compile, just include headers!

- Simple, light, and nothing else needed (like Boost).

- Supports different object lifetime management models.

- Convenient, type-safe access to the Lua stack.

- Automatic function parameter type binding.

- Does not require C++11.

## LuaBridge Demo and Tests

LuaBridge offers both a command line program, and a stand-alone graphical
program for compiling and running the test suite. The graphical program
provides an interactive window where you can enter execute Lua statements in a
persistent environment. The graphical program is cross platform and works on
Windows, Mac OS, iOS, Android, and GNU/Linux systems with X11. The stand-alone
program should work anywhere. Both of these applications include LuaBridge,
Lua version 5.2, and the code necessary to produce a cross platform graphic
application. They are all together in a separate repository, with no
additional dependencies, available here:

[LuaBridge Demo and Tests][4]

<a href=https://github.com/vinniefalco/LuaBridgeDemo>
<img src="https://github.com/vinniefalco/LuaBridgeDemo/blob/gh-pages/LuaBridgeDemoScreenshot.png">
</a><br>

<a href=https://github.com/vinniefalco/LuaBridgeDemo>
<img src="https://github.com/vinniefalco/LuaBridgeDemo/blob/gh-pages/OSBadges.png">
</a><br>

## Usage

LuaBridge is distributed as set of header files. You simply add 
`#include "LuaBridge.h"` where you want to register your functions, classes,
and variables. There are no additional source files, no compilation settings,
and no Makefiles or IDE-specific project files. LuaBridge is easy to
integrate.

LuaBridge provides facilities for taking C++ concepts like namespaces,
variables, functions, and classes, and exposing them to Lua. Because Lua is
weakly typed, the resulting structure is not rigid. The API is based on C++
template metaprogramming.  It contains template code to automatically
generate at compile-time the various Lua C API calls necessary to export your
program's classes and functions to the Lua environment.

There are four types of objects that LuaBridge can register:

- Data: Global varaibles, static class data members, and class data members.

- Functions: Regular functions, static class members, and class member
  functions

- Namespaces: A namespace is simply a table containing registrations of
  functions, data, properties, and other namespaces.

- Properties: Global properties, static class properties, and class member
  properties. These appear like data to Lua, but are implemented using get
  and set functions on the C++ side.

Both data and properties can be marked as _read-only_ at the time of
registration. This is different from `const`; the values of these objects can
be modified on the C++ side, but Lua scripts cannot change them. For brevity
of exposition, in the examples that follow the C++ code samples assume that a
`using namespace luabridge` declaration is in effect.

### Namespaces

All LuaBridge registrations take place in a _namespace_, which loosely
resembles a C++ namespace in Lua. When we refer to a _namespace_ we are
always talking about a namespace in the Lua sense, which is implemented using
tables. The Lua namespace does not need to correspond to a C++ namespace;
in fact no C++ namespaces need to exist at all unless you want them to.
LuaBridge namespaces are visible only to Lua scripts; they are used as a
logical grouping tool.

To obtain access to the global namespace for a `lua_State* L`
we call:

    getGlobalNamespace (L);

This returns a temporary object on which we can perform registrations that
go into the global namespace (a practice which is not recommended). Rather
than put multiple registrations into the global namespace, we can add a single
namespace to the global namespace, like this:

    getGlobalNamespace (L)
      .beginNamespace ("test");

The result is to create a table in `_G` (the global namespace in Lua) called
"test". Since we have not performed any registrations, this table will be mostly
empty, except for some necessary bookkeeping fields. LuaBridge reserves all
identifiers that start with a double underscore. So `__test` would be an
invalid name (although LuaBridge will silently accept it). Functions like
`beginNamespace` return the corresponding object on which we can make more
registrations. This:

    getGlobalNamespace (L)
      .beginNamespace ("test")
        .beginNamespace ("detail")
        .endNamespace ()
        .beginNamespace ("utility")
        .endNamespace ()
      .endNamespace ()
      ;

Creates the following nested tables: `_G["test"]`, `test["detail"]`, and
`test["utility"]`. The results are accessible to Lua as `test.detail` and
`test.utility`.
  
Here use the `endNamespace` function, which returns an object representing the
enclosing namespace, upon which further registrations may be added. It is
undefined behavior to use `endNamespace` on the global namespace. All
LuaBridge functions which create registrations return an object upon which
subsequent registrations can be made, allowing for an unlimited number of
registrations to be chained together using the dot operator `.`.

A namespace registration can be re-opened later to add more functions.
This lets you split up the registration between different source files.
The following are equivalent:

    getGlobalNamespace (L)
      .beginNamespace ("test")
        .addFunction ("foo", foo)
      .endNamespace ()
      ;

    getGlobalNamespace (L)
      .beginNamespace ("test")
        .addFunction ("bar", bar)
      .endNamespace ()
    ;

and

    getGlobalNamespace (L)
      .beginNamespace ("test")
        .addFunction ("foo", foo)
        .addFunction ("bar", bar)
      .endNamespace ()
    ;

Adding two objects with the same name, in the same namespace, results in
undefined behavior (although LuaBridge will silently accept it).

### Data, Properties, and Functions

Data, properties, and functions are registered into a namespace using
`addVariable`, `addProperty`, and `addFunction`. When functions registered
to Lua are called by Lua scripts, LuaBridge automatically takes care of the
conversion of arguments into the appropriate data type when doing so is
possible. This automated system works for the function's return value, and
up to 8 parameters although more can be added by extending the templates.
Pointers, references, and objects of class type as parameters are treated
specially, and explained in a later section.

Given the following:

    int globalVar;
    static float staticVar;
    std::string stringProperty;

    std::string getString () {
      return stringProperty;
    }
      
    void setString (std::string s) {
      return s;
    }

    int foo () {
      return 42;
    }

    void bar (char const*) {
    }

The will register everything into the namespace "test":

    getGlobalNamespace (L)
      .beginNamespace ("test")
        .addVariable ("var1", &globalVar)
        .addVariable ("var2", &staticVar, false)     // read-only
        .addProperty ("prop1", getString, setString)
        .addProperty ("prop2", getString)            // read only
        .addFunction ("foo", foo)
        .addFunction ("bar", bar)
      .endNamespace ()
      ;

Variables can be marked read-only by passing false in the second optional
parameter. If the parameter is omitted, `true` is used making the variable
read/write. Properties are marked read-only by omitting the set function or
passing it as 0 (or `nullptr`). After the registrations above, the following
Lua identifiers are valid:

    test        -- a namespace
    test.var1   -- a lua_Number variable
    test.var2   -- a read-only lua_Number variable
    test.prop1  -- a lua_String property
    test.prop2  -- a read-only lua_String property
    test.foo    -- a function returning a lua_Number
    test.bar    -- a function taking a lua_String as a parameter

Note that test.prop2 and test.prop1 both refer to the same value. However,
since test.prop2 is read-only, assignment does not work. These Lua statements
have the stated effects:

    test.var1 = 5         -- okay
    test.var2 = 6         -- error: var2 is not writable
    test.prop1 = "Hello"  -- okay
    test.prop1 = 68       -- okay, Lua converts the number to a string.
    test.prop2 = "bar"    -- error: prop2 is not writable

    test.foo ()           -- calls foo and discards the return value
    test.var1 = foo ()    -- calls foo and stores the result in var1
    test.bar ("Employee") -- calls bar with a string
    test.bar (test)       -- error: bar expects a string not a table

### Classes

A class registration is opened using either `beginClass` or `deriveClass` and
ended using `endClass`. Once registered, a class can later be re-opened for
more registrations using `beginClass`. However, `deriveClass` should only be
used once. To add more registrations to an already registered derived class,
use `beginClass`. We use the word _method_ as an unambiguous synonym for
_member function_, static or otherwise. Given:

    struct A {
      static int staticData;
      static float staticProperty;
        
      static float getStaticProperty () {
        return staticProperty;
      }

      static void setStaticProperty (float f) {
        staticProperty = f;
      }

      static void staticFunc () {
      }

      std::string dataMember;

      char dataProperty;

      char getProperty () const {
        return dataProperty;
      }

      void setProperty (char v) {
        dataProperty = v;
      }

      void func1 ()
      {
      }

      virtual void virtualFunc () {
      }
    };

    struct B : public A
    {
      double dataMember2;

      void func1 ()
      {
      }

      void func2 ()
      {
      }

      void virtualFunc () {
      }
    };

    int A::staticData;
    float A::staticProperty;

The corresponding registration statement is:

    getGlobalNamespace (L)
      .beginNamespace ("test")
        .beginClass <A> ("A")
          .addStaticData ("staticData", &A::staticData)
          .addStaticProperty ("staticProperty", &A::staticProperty)
          .addStaticMethod ("staticFunc", &A::staticFunc)
          .addData ("data", &A::dataMember)
          .addProperty ("prop", &A::getProperty, &A::setProperty)
          .addMethod ("func1", &A::func1)
          .addMethod ("virtualFunc", &A::virtualFunc)
        .endClass ()
        .deriveClass <B, A> ("B")
          .addData ("data", &B::dataMember2)
          .addMethod ("func1", &B::func1)
          .addMethod ("func2", &B::func2)
        .endClass ()
      .endClass ()
      ;

As with regular variables and properties, class data and properties can
be marked read-only by passing false in the second parameter, or omitting
the set function respectively. The `deriveClass` takes two template arguments:
the class to be registered, and its base class.  Inherited methods do not have
to be re-declared and will function normally in Lua. If a class has a base
class that is **not** registered with Lua, there is no need to declare it as a
subclass.

If we have objects of type A, and B, they can be passed to Lua using the
provided Stack interface. The following code example will be fully explained
in a later section, but for now it can be assumed that the effect is to
create values `_G["a"]` and `_G["b"]` which are userdata representing the
classes `A` and `B` respectively. After executing the following C++:

    A a;
    B b;

    Stack <A*>::push (L, &a);
    lua_setglobal (L, "a");
      
    Stack <B*>::push (L, &b);
    lua_setglobal (L, "a");

The following effects are observed in Lua scripts:

    print (test.A.staticData)       -- prints the static data member
    print (test.A.staticProperty)   -- prints the static property member
    test.A.staticFunc ()            -- calls the static method

    print (a.data)                  -- prints the data member
    print (a.prop)                  -- prints the property member
    a:func1 ()                      -- calls A::func1 ()
    test.A.func1 (a)                -- equivalent to a:func1 ()
    test.A.func1 ("hello")          -- error: "hello" is not a class A
    a:virtualFunc ()                -- calls A::virtualFunc ()

    print (b.data)                  -- prints B::dataMember
    print (b.prop)                  -- prints inherited property member
    b:func1 ()                      -- calls B::func1 ()
    b:func2 ()                      -- calls B::func2 ()
    test.B.func2 (a)                -- error: a is not a class B
    test.A.func1 (b)                -- calls A::func1 ()
    b:virtualFunc ()                -- calls B::virtualFunc ()
    test.B.virtualFunc (b)          -- calls B::virtualFunc ()
    test.A.virtualFunc (b)          -- calls B::virtualFunc ()
    test.B.virtualFunc (a)          -- error: a is not a class B

### Pointers, References, and Object Parameters

When C++ objects are passed from Lua back to C++ as arguments to functions,
or set as data members, LuaBridge does its best to automate the conversion.
Using the previous definitions, the following functions may be registered
to Lua:

    void func0 (A a);
    void func1 (A* a);
    void func2 (A const* a);
    void func3 (A& a);
    void func4 (A const& a);

Executing this Lua code will have the prescribed effect:

    func0 (a)   -- Passes a copy of a, using A's copy constructor.
    func1 (a)   -- Passes a pointer to a.
    func2 (a)   -- Passes a pointer to a const a.
    func3 (a)   -- Passes a reference to a.
    func4 (a)   -- Passes a reference to a const a.

In the example above, All functions can read the data members and property
members of `a`, or call const member functions of `a`. Only `func0`, `func1`
and `func3` can modify the data members and data properties, or call
non-const member functions of `a`.

Except for func0(), a problem can arise when C++ receives pointers or
references to objects seen by Lua. Undefined behavior can result if the Lua
garbage collector destroys the class while the C++ code is keeping a pointer
to the object somewhere. In the body of the function above, access is safe
because there is always a reference to the object on the Lua stack. To
prevent undefined behavior, do not store pointers to objects that come from
C++ functions called by Lua. Instead, use a _Container_. This is explained in
a later section.

The usual C++ inheritance and pointer assignment rules apply. Given:

    void func5 (B b);
    void func6 (B* b);

The following Lua effects are observed:

    func5 (b)   - Passes a copy of b, using B's copy constructor.
    func6 (b)   - Passes a pointer to b.
    func6 (a)   - Error: Pointer to B expected.
    func1 (b)   - Okay, b is a subclass of a.

When Lua manages the lifetime of the object, it is subjected to all of the
normal garbage collection rules. C++ functions and member functions can
receive pointers and references to these objects, but care must be taken
to make sure that no attempt is made to access the object after it is
garbage collected. Usually this is done by simply not storing a pointer to
the object somewhere inside your C++.

When C++ manages the lifetime of the object, the Lua garbage collector has
no effect on it. Care must be taken to make sure that the C++ code does not
destroy the object while Lua still has a reference to it.

### Data Types
  
Basic types for supported variables, and function arguments and returns, are:
  
- `bool`
- `char`, converted to a string of length one.
- Integers, `float`, and `double`, converted to Lua_number.
- Strings: `char const*` and `std::string`

Of course, LuaBridge supports passing objects of class type, in a variety of
ways including dynamically allocated objects created with `new`. The behavior
of the object with respect to lifetime management depends on the manner in
which the object is passed. Given `class T`, these argument types are
supported:

- `T`, `T const` : Pass `T` by value. The lifetime is managed by Lua.
- `T*`, `T&`, `T const*`, `T const&` : Pass `T` by reference. The lifetime
    is managed by C++.

Furthermore, LuaBridge supports a "shared lifetime" model: dynamically
allocated and reference counted objects whose ownership is shared by both
Lua and C++. The object remains in existence until there are no remaining
C++ or Lua references, and Lua performs its usual garbage collection cycle.
LuaBridge comes with a few varieties of containers that support this
shared lifetime model, or you can use your own (subject to some restrictions).

Mixing object lifetime models is entirely possible, subject to the usual
caveats of holding references to objects which could get deleted. For
example, C++ can be called from Lua with a pointer to an object of class
type; the function can modify the object or call non-const data members.
These modifications are visible to Lua (since they both refer to the same
object).

### Lua Stack

User-defined types which are convertible to one of the basic types are
possible, simply provide a `Stack <>` specialization in the `luabridge`
namespace for your user-defined type, modeled after the existing types.
For example, here is a specialization for a [juce::String][6]:

    template <>
    struct Stack <juce::String>
    {
      static void push (lua_State* L, juce::String s)
      {
        lua_pushstring (L, s.toUTF8 ());
      }

      static juce::String get (lua_State* L, int index)
      {
        return juce::String (luaL_checkstring (L, index));
      }
    };

## Limitations 

LuaBridge does not support:

- Enumerated constants
- More than 8 parameters on a function or method (although this can be
  increased by adding more `TypeList` specializations).
- Overloaded functions, methods, or constructors.
- Global variables (variables must be wrapped in a named scope).
- Automatic conversion between STL container types and Lua tables.
- Inheriting Lua classes from C++ classes.

## Development

[Github][3] is the new official home for LuaBridge. The old SVN repository is
deprecated since it is no longer used, or maintained. The original author has
graciously passed the reins to Vinnie Falco for maintaining and improving the
project. To obtain the older official releases, checkout the tags from 2.1
and earlier.

We welcome contributions to LuaBridge. Feel free to fork the repository and
issue a Pull Request. All questions, comments, suggestions, and/or proposed
changes will be handled by the new maintainer.

## License

Copyright (C) 2012, [Vinnie Falco][1] ([e-mail][0]) <br>
Copyright (C) 2007, Nathan Reed <br>
  
Portions from The Loki Library: <br>
Copyright (C) 2001 by Andrei Alexandrescu

License: The [MIT License][2]

Older versions of LuaBridge up to and including 0.2 are distributed under the
BSD 3-Clause License. See the corresponding license file in those versions
for more details.

[0]: mailto:vinnie.falco@gmail.com "Vinnie Falco (Email)"
[1]: http://www.vinniefalco.com "Vinnie Falco"
[2]: http://www.opensource.org/licenses/mit-license.html "The MIT License"
[3]: https://github.com/vinniefalco/LuaBridge "LuaBridge"
[4]: https://github.com/vinniefalco/LuaBridgeDemo "LuaBridge Demo"
[5]: http://lua.org "The Lua Programming Language"
[6]: http://www.rawmaterialsoftware.com/juce/api/classString.html "juce::String"
