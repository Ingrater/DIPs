| Title:          | making export an attribute                                         |
|-----------------|--------------------------------------------------------------------|
| DIP:            | 45                                                                 |
| Review Count:   | 0                                                                  |
| Author:         | Benjamin Thaut, Martin Nowak, David Nadlinger                      |
| Implementation: | TODO                                                               |
| Status:         | Draft                                                              |

Abstract
--------

Export and its behavior need to be changed in several ways to make it work on Windows and allow better code generation for other platforms. The Rationale section explains the problems and shows how this DIP solves them.

Description
-----------

Its highly recommended that you watch the Talk "D's import and export business" from DConf 2016 before reviewing this DIP. See the Links section.

-   The **export** protection level should be turned into an **export** attribute.
-   If a class/struct is annotated with the **export** attribute, all of its public and protected functions and members will automatically receive the **export** attribute. Also all its hidden compiler specific symbols will receive the **export** attribute.
-   A new -useShared windows only flag is added to dmd roughly comparable to -fPIC on linux.
-   All D shared libraries built on windows must link against the shared version of phobos. The only exception from this rule is a single dll which does not intend to share any memory with other D programs.

Rationale
---------

### Turning export into an attribute

Currently **export** is a protection level, the highest level of visibility. This however conflicts with the need to export 'protected' and 'private' symbols. Consider a Base class in a shared library.

``` D
module sharedLib;

class Base {
  protected final void doSomething() { ... }
}
```

``` D
module executable;
import sharedLib;

class Derived : Base {
  public void func()
  {
    doSomething();
  }
}
```

In the above example 'doSomething' should only be visible to derived classes but it still needs to be exportable from a shared library. Therefor **export** should become a normal attribute which behaves orthogonal to protection.

Also consider the following example in which the template will access a private function. Because the template is instantiated on the user side and not within the shared library it is required to export the private function so that the template can access it from outside the shared library.

``` D
module dll;

void copy(T)(T val)
{
  copyImpl(&val, T.sizeof);
}

export private copyImpl(void* mem, size_t size)
{
  ...
}
```

``` D
module exe;
import dll;

void main(string[] args)
{
  int bar = 0;
  copy(bar); // template will be instantiated in the exe but needs access to the copyImpl function.
}
```

Another special case are voldemort types. If a type, that is declared within a function, is returned from that function and this function is **export** the type itself will also be exported.

``` D
module lib;

export auto makeSomething(int v)
{
  struct Something // Exported automatically.
  {
    int i;
  }

  return Something(v);
}
```

As the type declarations in a function can get more complex, it is still possible to explicitly export inner types.
``` D
module lib;

export auto makeSomething(int v)
{
  // not exported automatically because not returned from the function
  // so export explicitly.
  export struct SomethingElse
  {
	float f;
  }

  struct Something // Exported automatically.
  {
    int i;
	SomethingElse* somethingElse;
  }

  return Something(v);
}
```

At first glance exporting a template doesn't make much sense. But consider the following example:

``` D
module lib;

import std.stdio;

export struct Foo(T)
{
  T value;
  void print() { writefln("%s", value); }
}

__gshared Foo!int g_inst = Foo!int(5);
```

``` D
module exe;
import lib;

void main(string[] args)
{
  auto f = Foo!int(5);
  f.print();
}
```

When compiling the executable module exe which uses the module lib compiled into a dll the compiler will attempt to reuse the instance of Foo!int from the lib module. This however only works if the instance has been exported from the dll. As a result exporting a template should be equivalent to exporting any instance created from this template. In code:

``` D
export struct Foo(T) { ... }

// is equivalent to
template Foo(T)
{
  export struct Foo { ... }
}
```

### export attribute inference

Currently export has to be specified in a lot of places to export all necessary functions and data symbols. Export should be transitive for aggregate types (structs/classes) so that when exporting a aggregate type export is applied to all public & protected members without the need to add export to every single public and protected member.

``` D
module sharedLib;

export class A                          // compiler internal members should be exported (e.g. vtable, type info)
{
  private:
    int m_a;

    static int s_b;              // should not be exported (because TLS see "Access TLS variables")

    void internalFunc() { ... }  // should not be exported

  protected:
    void interalFunc2() { ... }  // should be exported

  public:
    class Inner                  // compiler internal members should be exported
    {
      __gshared int s_inner;            // should be exported

      void innerMethod() { ... } // should be exported
    }

    void method() { ... }        // should be exported
}
```

### Access TLS variables

Currently it is not possible to access TLS variables across shared library boundaries on windows. The user is expected to wrap access to TLS variables into a property. E.g.

Before:
``` D
int tlsCounter = 0;
```

After:
``` D
int _tlsCounter = 0;
export auto tlsCounter() @property
{
    // if this is inlined we are on the wrong side of the dll boundary
    pragma(inline, false);
    return _tlsCounter;
}

export void tlsCounter(int value) @property
{
    pragma(inline, false);
    _tlsCounter = value;
}
```

If a TLS variable is annotated with export no compiler error will be given to allow the following.

``` D
export:

__gshared int g_var1; // will be exported
shared(int) g_var2; // will be exported
int g_var3; // will not be exported because TLS

class Foo // will be exported
{
   ...
}
```

Issuing a compiler error when attempting to export an TLS variable would result in the need to take all TLS variables out of export blocks.
Accessing a TLS variable across a shared library boundary will however always result in a linker error.

### -useShared compiler flag

Dmd will receive a new -useShared compiler flag which is windows only and serves the following purpose:
- Code generation will assume that it generates code which either uses other D-dlls or will end up in a D-dll itself. This will result in additional indirections and slower code compared to static linking. As a result this should not be always on. Loosely comparable with -fPIC on linux. The difference though is, that -useShared needs to be specified on the executable that uses other D-dlls and  when compiling dlls. With this flag only executables that use dynamic linking will suffer a small performance impact. Statically linked executables will not be affected. If this flag is not added to dmd, all executables on windows would suffer some of the performance overhead of dynamic linking, even fully statically linked ones.
- With -useShared specified the compiler will generate additional data to allow for the relocation of data symbols across dll boundaries.
- The compiler will automatically link against the shared version of phobos.

Summary:
-useShared enables importing of symbols from dlls. Required for executables that use shared libraries written in D and all D-dlls that link against the shared version of phobos.
-shared enables exporting of symbols from dlls. Required for shared libraries written in D.

### Example

``` D
module lib1;

version(Shared)
{
    import core.sys.windows.dll;

    // This dll is going to be used from D code. 
    // Which means we don't have to hijack all threads in 
    // existence as we have a shared druntime that knowns 
    // about all D threads already.
    mixin SimpleDllMain!(DllIsUsedFromC.no);
}

export class LibClass
{
public:
    void doSomething() { ... }
}
```

``` D
module lib2;

export struct LibStruct
{
    void doSomething() { ... }
}
```

``` D
module exe;
import lib1;
import lib2;

void main(string[] args)
{
    auto inst = new LibClass();
    inst.doSomething();
    
    LibStruct s;
    s.doSomething();
}
```

Compiling into shared library and executable (e.g. dynamic linking):
```
dmd -m64 -shared lib1.d lib2.d -version=Shared -ofmylib.dll -L/IMPLIB:mylib.lib
dmd -m64 -useShared exe.d mylib.lib -ofexe.exe
```
Note that both mylib.dll and exe.exe will link against the shared version of phobos, phobos64s.dll. As a result both executables will share the same druntime, and therefore the same gc, type data etc.
Garbage collection will just work without the need to set a GC proxy, throwing exceptions across dll boundaries will work and all other things you would come to expect from a shared library implementation.


Compiling into statically linked executable:
```
dmd -m64 lib1.d lib2.d exe.d -ofexe.exe
```

Changes to the D Grammar
----------------------

**export** is removed as a visibility attribute.

```
VisibilityAttribute:
    private
    package
    package ( IdentifierList )
    protected
    public
```

**export** is added as regular attribute.

```
Attribute:
    LinkageAttribute
    AlignAttribute
    DeprecatedAttribute
    VisibilityAttribute
    Pragma
    static
    extern
    abstract
    final
    override
    synchronized
    auto
    scope
    const
    immutable
    inout
    shared
    __gshared
    Property
    nothrow
    pure
    ref
    return
    export
```

**export** is now a valid function attribute:

```
FunctionAttribute:
    nothrow
    pure
    Property
    export
```

**export** is added as storage class:

```
StorageClass:
    LinkageAttribute
    AlignAttribute
    deprecated
    enum
    static
    extern
    abstract
    final
    override
    synchronized
    auto
    scope
    const
    immutable
    inout
    shared
    __gshared
    Property
    nothrow
    pure
    ref
    export
```

### Breaking Change

This change to the grammar is a breaking change. It is possible that D programs that compiled before this change no longer compile after this change. This especially happens when **export** truly served as a protection level:

``` D
private:
export void func() { ... }
```

With the current version of **export** as protection attribute the function would turn public due to **export**. After the change the function will have private protection as export no longer serves as a protection attribute and any call to func outside of the module will result in a access denied error. Research on all git hub projects that use D has however shown that **export** is not used anyway and in the rare cases it is used it would work the same way before and after these changes. So breakage should be kept to a minimum.


Implementation Details
----------------------

### Windows

#### The basic problem with importing and exporting symbols

First lets take a look at how C/C++ solve this problem. MSVC has two attributes for exporting and importing symbols respectively \_\_declspec(dllimport) and \_\_declspec(dllexport).
Usually they are hidden behind a preprocessor macro like this:

``` C
#ifdef LIB_AS_DLL
    #ifdef LIB_EXPORTS
        #define LIB_API __declspec(dllexport)
    #else
        #define LIB_API __declspec(dllimport)
    #endif
#else
	#define LIB_API
#endif
```

Each library has its own LIB_API macro.

These preprocessor defines are set in different situations:
When compiling the library into a static library: none
When using the library as a static library: none
When compiling the library into a shared library: LIB_AS_DLL, LIB_EXPORTS
When using the library as a shared library: LIB_AS_DLL

Lets assume we have two libraries A and B with two macros A_API and B_API respectively.

a.h + b.h combined will look like this:
``` C
A_API void funcA();
B_API void funcB();
```

When compiling statically and using the library statically the C/C++ compiler will always see the following:
``` C
// Defines: none
void funcA();
void funcB();
```

When compiling library A into a shared library the compiler will see:
``` C
// Defines: LIB_A_AS_DLL, LIB_A_EXPORTS, LIB_B_AS_DLL
__declspec(dllexport) void funcA();
__declspec(dllimport) void funcB();
```

When compiling library B into a shared library the compiler will see:
``` C
// Defines: LIB_A_AS_DLL, LIB_B_AS_DLL, LIB_B_EXPORTS
__declspec(dllimport) void funcA();
__declspec(dllexport) void funcB();
```

When compiling the executable that uses library A and library B the compiler will see:
``` C
// Defines: LIB_A_AS_DLL, LIB_B_AS_DLL
__declspec(dllimport) void funcA();
__declspec(dllimport) void funcB();
```

As you can see in C/C++ it is always clearly defined if the symbol must be imported / exported or if it is a "regular" symbol.

In D the same situation looks like this:

libA.d
``` D
module libA;
export void funcA();
```

libB.d
``` D
module libB;
export void funcB();
```

And this is what the compiler will see in all of the above cases. There is no indication in source code if the symbol must be exported / imported or if its a "regular" symbol.
This section will describe how this problem is solved.

#### Importing and Exporting
To decide whether or not a symbol should be exported the compiler has to check the following conditions. This is the equivalent of \_\_declspec(dllexport) in the msvc compiler.

1) Does the symbol belong to a module that is a root module (part of the current compilation?).
2) Is the symbol or its surrounding aggregate type declared as export? 
3) Do we currently compile a dll (-shared specified on command line)

If all conditions are met the symbols are exported.

To decide whether a symbol must be imported the compiler has to check the following conditions. This is the equivalent of \_\_declspec(dllimport) in the msvc compiler.
1) Does the symbol belong to a module that is not a root module?
2) Is the symbol or its surrounding aggregate type declared as export? 
3) Is the -useShared flag set?

If all of these conditions met the symbol must be imported. Imported symbols are not referred to directly but instead use one level of indirection to allow the operating system loader to patch in the actual address of the symbol at executable load time. Msvc uses the prefix \_\_imp\_ for so called import symbols. These symbols are generated by the linker automatically into the import library.
Note that the conditions 1 and 2 are conservative. In some situations a symbol will be imported even though it is part of the same binary. For example with multi object compilation or when linking in static libraries. This will result in suboptimal code as additional unnecessary indirections are inserted. To prevent the slow down of fully statically linked D programs the third condition is introduced. This way the additional indirections will only be introduced when building with shared libraries.

The following flags will be passed to the D compiler:
When compiling the library into a static library: -lib
When using the library as a static library: none
When compiling the library into a shared library: -shared -useShared
When using the library as a shared library: -useShared

See the Examples section for detailed usage examples.

#### Special case template

In a binary sense templates don't belong to the module they are defined in. Consider the following example:

``` D
module lib;

export private void someFuncImpl(string typeName)
{
    // implementation detail
}

export void someFunc(T)()
{
    someFuncImpl(T.stringof);
}
```

``` D
module exe;
import lib;

void main(string[] args)
{
    someFunc!int();
}
```

When the shared library mylib will be compiled the code for the template "someFunc" will not be generated as it is not yet known what instances are needed. So the shared library mylib.dll will not contain any code for someFunc.
When compiling the executable the template instance someFunc!int is required and thus instanciated. The code will be generated and be part of the executable. This means in a binary sense the template instance someFunc!int does not belong to the shared library but instead belongs to the executable. This affects if the function someFunc!int() should be exported/imported. As a result when deciding if a symbol should be exported/imported and the symbol is part of a template the compiler won't use the module the template is defined in. Instead the compiler will use the module that caused the instantiation of the template as basis for the export/import decision.

#### Referencing in constant data

When referencing data symbols in the contents of some other data symbol there will be a additional level of indirection which needs to be removed during program startup.

``` D
module dll;

__gshared int var = 5;
```

``` D
module exe;
import dll;

__gshared int* pvar = &var; // address not known at compile time or link time
```

As the address of var is not known at compile time so pvar will point to the entry in the import table for 'var'. At program startup, before any other D code runs, pvar will be dereferenced once. E.g. the following operation will be executed on pvar.

``` D
pvar = *cast(int**)pvar;
```

This removes the additional indirection added by the import table and correctly initializes the static memory for pvar. This might happen in various other cases, mostly when generating initializers, type infos, vtables, module infos and other static data the compiler produces.

Examples
--------

### Example 1: Hello World!

Compile a simple hello world program against the shared version of phobos.

hello.d:
``` D
module hello;

import std.stdio;

void main(string[] args)
{
	writefln("Hello World!");
}
```

Compile and execute with:
dmd -m64 -useshared hello.d

### Example 2: First shared library

Compile a simple module into a shared library and use it.

lib.d:
``` D
module lib;

version(Shared)
{
	import core.sys.windows.dll;

	// This dll is going to be used from D code. 
	// Which means we don't have to hijack all threads in 
	// existence as we have a shared druntime that knowns 
	// about all D threads already.
	mixin SimpleDllMain!(DllIsUsedFromC.no);
}

export class LibClass
{
public:
	void doSomething() 
	{
		import std.stdio;
		writefln("Something");
	}
}
```

program.d:
``` D
module program;
import lib;

void main(string[] args)
{
	auto inst = new LibClass();
	inst.doSomething();
}
```

Compile into a dll and executable:
```
dmd -m64 -shared -useshared -version=Shared lib.d -L/IMPLIB:lib-dynamic.lib
dmd -m64 -useshared program.d lib-dynamic.lib -ofprogram-dynamic.exe
```

Compile into a single statically linked executable:
```
dmd -m64 lib.d -lib -oflib-static.lib
dmd -m64 program.d lib-static.lib -ofprogram-static.exe
```

### Example 3: Link a static library into a shared library

libA.d:
``` D
module libA;

import std.stdio;

void print(string s)
{
  writeln(s);
}
```

libB.d:
``` D
module libB;

import libA;
import core.sys.windows.dll;

mixin SimpleDllMain!(DllIsUsedFromC.no);

export void printFwd(string s)
{
    print(s);
}
```

program.d
``` D
module program;

import libB;
import std.stdio;

void main(string[] args)
{
    printFwd("Hello World!");
}
```

Compile and link into one shared library and one program that uses it.
```
dmd -m64 -lib -useshared libA.d -oflibA.lib
dmd -m64 -useshared -shared libB.d libA.lib -oflibB.dll -L/IMPLIB:libB.lib
dmd -m64 -useshared program.d libB.lib
```

Open Problems
-------------

Problems related to shared libraries on Windows not addressed by this DIP are:
- The shared version of phobos will link against a specific version of the C-runtime. As the C-runtimes are not compatible with each other the C-runtime linked into the executable must match the version linked into the shared phobos dll. Thus the question is, which version of the shared phobos dll will we ship with the dmd installer on windows? Ship multiple version to support all Visual Studio version? Will we only ship one version linked against the latest Visual Studio? Will we build the shared version of phobos during install?
- How will we test that the set of export annotations is complete? e.g. a user of a library will not encounter linker errors due to a missing **export**? The current D unittests are not suiteable to do this as they will end up in the same executable as the code to be tested. But to properly test exporting of symbols the test code most reside on the other side of the shared library boundary.
- Cross shared library boundary inlining: I couldn't decide on a good policy here. Due to not supporting TLS access functions which use TLS can't be inlined. But everything else could be inlined. Inlining functions accross shared library boundaries however means, that functions can't be updated by supplying a new version of the shared library.

Future Work
-----------

Once this DIP is implemented the following items should be straight forward to do.
- Change the default symbol visibility on *nix to hidden and only export symbols from shared library which have the **export** attribute. This will have a positive performance impact on *nix shared libraries as the compiler is able to inline more function calls and the operating system load has to process less symbols on shared library load.
- Implement shared library support on mac os.
- Go through all of druntime and phobos and add **export** to all necessary symbols.

Links
-----
- [DConf 2016 Talk](https://youtu.be/MQRHxI2SrYM)
- [Getting started with D-Dlls](http://stuff.benjamin-thaut.de/D/getting_started.html)
- [DIP45: fixing the dllimport/dllexport issue](http://forum.dlang.org/post/kvhu2c$2ikq$1@digitalmars.com)
- [Issue 9816 – Export is mostly broken](https://issues.dlang.org/show_bug.cgi?id=9816)
- [Issue 4071 - Missing support to share memory and objects between DLLs and executable](https://issues.dlang.org/show_bug.cgi?id=4071)
- [Issue 7020 - Exception thrown across DLL is not caught](https://issues.dlang.org/show_bug.cgi?id=7020)

Copyright
---------

This document has been placed in the Public Domain.
