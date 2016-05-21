
# Wren++

A C++ wrapper for the [Wren programming language](http://munificent.github.io/wren/). Both Wren and this library are still under development, so not everything here is stable yet.

Wren++ currently provides:
- A RAII wrapper for the Wren virtual machine
- Automatic binding code generation for any C++ function and class
- Convenient access for calling Wren class methods from C++
- Template-based -- no macros!

Current deficiencies:
- Not type-safe. It's undefined what happens when you try to bind code that returns a type which hasn't itself been bound (most likely a crash is going to happen)
- Wren++ has no concept of const-ness. When you pass an object via const-pointer to Wren, the resulting Wren foreign object will happily call the non-const methods of the object -- yikes!
- Wren access from C++ is rather minimal

Currently developing against `wren:master@0a060a9`. [![Build Status](https://travis-ci.org/Nelarius/wrenpp.svg?branch=master)](https://travis-ci.org/Nelarius/wrenpp)

## Table of contents
* [Build](#build)
* [At a glance](#at-a-glance)
* [Accessing Wren from Cpp](#accessing-wren-from-cpp)
  * [Methods](#methods)
* [Accessing Cpp from Wren](#accessing-cpp-from-wren)
  * [Foreign methods](#foreign-methods)
  * [Foreign classes](#foreign-classes)
    * [Properties](#properties)
    * [Methods](#methods)
  * [CFunctions](#cfunctions)
  * [Cpp and Wren lifetimes](#cpp-and-wren-lifetimes)
* [Customize VM behavior](#customize-vm-behavior)
  * [Customize printing](#customize-printing)
  * [Customize error printing](#customize-error-printing)
  * [Customize module loading](#customize-module-loading)
  * [Customize heap allocation and garbage collection](#customize-heap-allocation-and-garbage-collection)
  * [Known issues](#known-issues)

## Build

Clone the repository using `git clone https://github.com/nelarius/wrenpp.git`. The easiest way to build the project is to include the contents of the `src/` folder in your project, since there's so little code. Just remember to compile with C++14 features turned on!

Alternatively, you can build the static library with premake:

```sh
premake5 vs2015 --include=<path to wren.h>
```

If you want to build the tests as well, then you need to include the location of wren's `lib/` folder.

```sh
premake5 vs2015 --include=<path to wren.h> --link=<path to wren/lib>
``` 

## At a glance

Let's fire up an instance of the Wren VM and execute some code:

```cpp
#include "Wren++.h"

int main() {
  wrenpp::VM vm{};
  vm.executeString( "System.print(\"Hello, world!\")" );

  return 0;
}
```

The virtual machine is held internally in a pointer. `Wren` uniquely owns the virtual machine, which means that the `Wren` instance can't be copied, but can be moved when needed - just like a unique pointer.

Module names work the same way by default as the module names of the Wren command line module. You specify the location of the module without the `.wren` postfix.

```cpp
vm.executeModule( "script" );   // refers to script.wren
```

Both `executeString` and `executeModule` return a code indicating if the interpretation encountered any errors.

```cpp
wrenpp::Result res = vm.executeString(
  "// calling nonexistent variable"
  "foobar.call()"
);
if ( res == wrenpp::Result::CompileError ) {
  std::cout << "foobar doesn't exist and a compilation error occurs.\n";
}
```

There are two other codes: `wrenpp::Result::RuntimeError` and `wrenpp::Result::Success`.

## Accessing Wren from Cpp

### Methods

You can use the `Wren` instance to get a callable handle for a method implemented in Wren. Given the following class, defined in `bar.wren`, 

```dart
class Foo {
  static say( text ) {
    System.print( text )
  }
}
```

you can call the static method `say` from C++ by using `void Method::operator( Args&&... )`,

```cpp
wrenpp::VM vm{};
vm.executeModule( "bar" );

wrenpp::Method say = vm.method( "main", "Foo", "say(_)" );
say( "Hello from C++!" );
```

`VM::method` has the following signature:

```cpp
Method VM::method( 
  const std::string& module, 
  const std::string& variable,
  const std::string& signature
);
```

`module` will be `"main"`, if you're not in an imported module. `variable` should contain the variable name of the object that you want to call the method on. Note that you use the class name when the method is static. The signature of the method has to be specified, because Wren supports function overloading by arity (overloading by the number of arguments).

The return value of a Wren method can be accessed by doing

```cpp
wrenpp::Method returnsThree = vm.method("main", "returnsThree", "call()");
wrenpp::Value val = returnsThree();
double val = val.as<double>();
```

The `operator()` method on `wrenpp::Method` returns a `wrenpp::Value` object, which can be cast to the wanted return type by calling `as<T>()`.

Number, boolean, and string values are stored within `wrenpp::Value` itself. Note that do to this, you *don't* want to write

```cpp
const char* greeting = returnsGreeting().as<const char*>();
```

because the string value is storesd in the `wrenpp::Value` instance itself. This would result in trying to dereference a string value which no longer exists. Store it locally before using the string value, like we did above:

```cpp
wrenpp::Value greeting = returnsGreeting();
printf("%s\n", greeting.as<const char*>());
```

## Accessing Cpp from Wren

**TODO:** improve

Wren++ allows you to bind C++ functions and methods to Wren classes. This is done by storing the names or signatures along with the function pointers, so that Wren can find them. That is, you bind all your code once, after which all your Wren instances can use them.

### Foreign methods

You can implement a Wren foreign method as a stateless free function in C++. Wren++ offers an easy to use wrapper over the functions. Note that only primitive types and `std::string` work for now. Support for registered types will be provided later once the custom type registration feature is complete.

Here's how you could implement a simple math library in Wren by binding the C++ standard math library functions.

math.wren:
```dart
class Math {
  foreign static cos( x )
  foreign static sin( x )
  foreign static tan( x )
  foreign static exp( x )
}
```
main.cpp:
```cpp
#include "Wren++.h"
#include <cmath>

int main() {

  wrenpp::beginModule( "math" )
    .beginClass( "Math" )
      .bindFunction< decltype(&cos), &cos >( true, "cos(_)" )
      .bindFunction< decltype(&sin), &sin >( true, "sin(_)" )
      .bindFunction< decltype(&tan), &tan >( true, "tan(_)" )
      .bindFunction< decltype(&exp), &exp >( true, "exp(_)" )
    .endClass()
  .endModule();

  wrenpp::VM vm;
  vm.executeString( "import \"math\" for Math\nSystem.print( Math.cos(0.12345) )" );

  return 0;
}
```

Both the type of the function (in the case of `cos` the type is `double(double)`, for instance, and could be used instead of `decltype(&cos)`) and the reference to the function have to be provided to `bindFunction` as template arguments. As arguments, `bindFunction` needs to be provided with a boolean which is true, when the foreign method is static, false otherwise. Finally, the method signature is passed.

> The free function needs to call functions like `wrenGetArgumentDouble`, `wrenGetArgumentString` to access the arguments passed to the method. When you register the free function, Wren++ wraps the free function and generates the appropriate `wrenGetArgument*` function calls during compile time. Similarly, if a function returns a value, the call to the appropriate `wrenReturn*` function is inserted at compile time.

### Foreign classes

Free functions don't get us very far if we want there to be some state on a per-object basis. Foreign classes can be registered by using `bindClass` on a module context. Let's look at an example. Say we have the following Wren class representing a 3-vector:

```dart
foreign class Vec3 {
  construct new( x, y, z ) {}

  foreign norm()
  foreign dot( rhs )
  foreign cross( rhs )    // returns the result as a new vector

  // accessors
  foreign x
  foreign x=( rhs )
  foreign y
  foreign y=( rhs )
  foreign z
  foreign z=( rhs )
}
```

We would like to implement it using the following C++ struct.

```cpp
struct Vec3 {
  union {
    float v[3];
    struct { float x, y, z; };
  };
  
  Vec3( float x, float y, float z )
  : v{ x, y, z }
    {}

  float norm() const {
    return sqrt( x*x + y*y + z*z );
  }

  float dot( const Vec3& rhs ) const {
    return x*rhs.x + y*rhs.y + z*rhs.z;
  }

  Vec3 cross( const Vec3& rhs ) const {
    return Vec3 {
      y*rhs.z - z*rhs.y,
      z*rhs.x - x*rhs.z,
      x*rhs.y - y*rhs.x
    };
  }
};
```

Let's start by binding the class to Wren and adding properties.

A class is bound by writing `bindClass` instead of `beginClass`. This binds the specified constructor and destructor to Wren, and allocates a new instance of your class within Wren.

```cpp
#include "Wren++.h"

int main() {
  wrenpp::beginModule( "main" )
    .bindClass< Vec3, float, float, float >( "Vec3" );
    // you can now construct Vec3 in Wren

  return 0;
}
```

Pass the class type, and constructor argument types to `bindClass`. Even though a C++ class may have many constructors, only one constructor can be registered with Wren.

#### Properties

If your class or struct has public fields you wish to expose, you can do so by using `bindGetter` and `bindSetter`. This will automatically generate a function which returns the value of the field to Wren.

```cpp
wrenpp::beginModule( "main" )
  .bindClass< Vec3, float, float, float >( "Vec3" )
    .bindGetter< decltype(&Vec3::x), &Vec3::x >( "x" )
    .bindSetter< decltype(&Vec3::x), &Vec3::x >( "x=(_)" )
    .bindGetter< decltype(&Vec3::y), &Vec3::y >( "y" )
    .bindSetter< decltype(&Vec3::y), &Vec3::y >( "y=(_)" )
    .bindGetter< decltype(&Vec3::z), &Vec3::z >( "z" )
    .bindSetter< decltype(&Vec3::z), &Vec3::z >( "z=(_)" );
```

Getters and setters are implicitly assumed to be non-static methods.

#### Methods

Using `registerMethod` allows you to bind a class method to a Wren foreign method. Just do:

```cpp
#include "Wren++.h"

int main() {
  wrenpp::beginModule( "main" )
    .bindClass< Vec3, float, float, float >( "Vec3" )
      // properties
      .bindGetter< decltype(&Vec3::x), &Vec3::x >( "x" )
      .bindSetter< decltype(&Vec3::x), &Vec3::x >( "x=(_)" )
      .bindGetter< decltype(&Vec3::y), &Vec3::y >( "y" )
      .bindSetter< decltype(&Vec3::y), &Vec3::y >( "y=(_)" )
      .bindGetter< decltype(&Vec3::z), &Vec3::z >( "z" )
      .bindSetter< decltype(&Vec3::z), &Vec3::z >( "z=(_)" )
      // methods
      .bindMethod< decltype(Vec3::norm), &Vec3::norm >( false, "norm()" )
      .bindMethod< decltype(Vec3::dot), &Vec3::dot >( false, "dot(_)" )
    .endClass()
  .endModule();

  return 0;
}
```

The arguments are the same as what you pass `bindFunction`, but as the template parameters pass the method type and pointer instead of a function.

We've now implemented two of `Vec3`'s three foreign functions -- what about the last foreign method, `cross(_)` ?

### CFunctions

Wren++ let's you bind functions of the type `WrenForeignMethodFn`, typedefed in `wren.h`, directly. They're called CFunctions for brevity (and because of Lua). Sometimes it's convenient to wrap a collection of C++ code manually. This happens when the C++ library interface doesn't match Wren classes that well. Let's take a look at binding the excellent [dear imgui](https://github.com/ocornut/imgui) library to Wren.

Many functions to ImGui are very overloaded and have lengthy signatures. We would have to fully qualify the function pointers, which would make the automatic bindings a mess. Additionally, many ImGui functions (such as SliderFloat) take in pointers to primitive types, like float, bool. Wren doesn't have any concept of `out` parameters, so we will make our Wren ImGui API take in a number by value, and return the new value.

```dart
class Imgui {

  // windows
  foreign static begin( name )    // begin a window scope
  foreign static end()            // end a window scope

  foreign static sliderFloat( label, value, min, max )  // RETURNS the new value
}
```

First, let's implement wrappers for `ImGui::Begin` and `ImGui::SliderFloat` with reasonable default values.

```cpp
void begin(WrenVM* vm) {
  ImGui::Begin((const char*)wrenGetSlotString(vm, 1), NULL, 0);
}

void sliderFloat(WrenVM* vm) {
  const char* label = wrenGetSlotString(vm, 1);
  float value = float(wrenGetSlotDouble(vm, 2));
  float min =   float(wrenGetSlotDouble(vm, 3));
  float max =   float(wrenGetSlotDouble(vm, 4));
  ImGui::SliderFloat(label, &value, min, max);
  wrenSetSlotDouble(vm, 0, value);
}
```

Here's what the binding code looks like. Note that ImGui::End is trivial to bind as it takes no arguments.

```cpp
wrenpp::beginModule( "builtin/imgui" )
  .beginClass( "Imgui" )
    // windows & their formatting
    .bindCFunction( true, "begin(_)", VM::begin )
    .bindFunction< decltype(&ImGui::End), &ImGui::End>( true, "end()" )
    .bindCFunction( true, "sliderFloat(_,_,_,_)", VM::sliderFloat)
  .endClass();
```

If you need to access and return foreign object instances within your CFunction, you can use the following two helper functions.

Use `wrenpp::getSlotForeign<T>(WrenVM*, int)` to get a bound type from the slot API:

```cpp
void setWindowSize(WrenVM* vm) {
  ImGui::SetNextWindowSize(*(const ImVec2*)wrenpp::getSlotForeign<Vec2i>(vm, 1));
}
```

Use `wrenpp::setSlotForeignValue<T>(WrenVM*, int, const T&)` and `wrenpp::setSlotForeignPtr<T>(WrenVM*, int, T* obj)` to place an object with foreign bytes in a slot, by value and by reference, respectively. `wrenpp::setSlotForeignValue<T>` uses the type's copy constructor to copy the object into the new value.

### Cpp and Wren lifetimes

If the return type of a bound method or function is a reference or pointer to an object, then the returned wren object will have C++ lifetime, and Wren will not garbage collect the object pointed to. If an object is returned by value, then a new instance of the object is also constructed withing the returned Wren object. In this situation, the returned Wren object has Wren lifetime and is garbage collected.

## Customize VM behavior

### Customize printing

You can provide your own implementation for `System.print` by assigning a callable object with the signature `void(WrenVM*, const char*)` to `wrenpp::VM::writeFn`. By default, `writeFn` is implemented as follows:

```cpp
VM::writeFn = []( WrenVM* vm, const char* text ) -> void { printf( text ) };
```

### Customize error printing

You can provide your own function to route error messages. Assign a callable object with the signature `void(WrenErrorType, const char*, int, const char*)` (see wren.h for the signature of `WrenErrorFn`) to `wrenpp::VM::errorFn`. By default, Wren++ prints errors to `stdout` as

```
WREN_ERROR_COMPILE in main:15 > Error at 'Vec3': Variable is used but not defined.
```

### Customize module loading

When the virtual machine encounters an import statement, it executes a callback function which returns the module source for a given module name. If you want to change the way modules are named, or want some kind of custom file interface, you can change the callback function. Just set give `VM::loadModuleFn` a new value, which can be a free standing function, or callable object of type `char*( const char* )`.

By default, `VM::loadModuleFn` has the following value.

```cpp
VM::loadModuleFn = []( const char* mod ) -> char* {
    std::string path( mod );
    path += ".wren";
    auto source = wrenpp::fileToString( path );
    char* buffer = (char*) malloc( source.size() );
    memcpy( buffer, source.c_str(), source.size() );
    return buffer;
};
```

### Customize heap allocation and garbage collection

You can use your own allocation/free functions assigning them to the `wrenpp::VM::allocateFn` and `wrenpp::VM::freeFn` callbacks. `allocateFn` must be a callable with the `void*(std::size_t)` signature, and `freeFn` must be a callable with the `void(void*)` signature. By default, the callbacks are just set to the system malloc/free:

```cpp
wrenpp::VM::allocateFn = std::malloc;

wrenpp::VM::freeFn = std::free;
```

Wren++ uses the allocation and free function to allocate large chunks of memory at a time. The `chunkSize` parameter specifies how much memory is reserved at a time, in bytes. The default value for the block size is 5 MiB.

`wrenpp::VM::chunkSize = 0x500000u;`

The purpose of allocating memory chunks is to avoid expensive system calls for allocating memory, if you're using the default malloc and free. Wren++ contains a small memory manager, which emulates malloc/free/realloc within the memory chunks. These are the functions that are actually passed to Wren as callbacks. The memory manager tries to keep fragmentation low by always allocating blocks of memory which are powers of two in size. This means that the blocks are frequently larger than necessary, but it also means that realloc calls often don't require a memory move. As a result, Wren usually runs slightly faster (around 5%) under Wren++. Currently, the memory manager uses a first-fit policy to determine which free memory blocks are to be reused.

The initial heap size is the number of bytes Wren will have allocated before triggering the first garbage collection. By default, it's 10 MiB.

`wrenpp::VM::initialHeapSize = 0xA00000u;`

After a collection occurs the heap will have shrunk. Wren will allow the heap to grow to (100 + heapGrowthPercent) % of the current heap size before the next collection occurs. By default, the heap growth percentage is 50 %.

`wrenpp::VM::heapGrowthPercent = 50;`

The minimum heap size is the heap size, in bytes, below which collections will not be carried out. The idea of the minimum heap size is to avoid miniscule heap growth (calculated based on the percentage mentioned previously) and thus very frequent collections. By default, the minimum heap size is 1 MiB.

`wrenpp::VM::minHeapSize = 0x100000u;`

### Known issues

* Calling Wren methods from C++, which themselves call foreign methods, doesn't work currently. This is due to a known [issue](https://github.com/munificent/wren/issues/362).

## TODO:

* A compile-time method must be devised to assert that a type is registered with Wren. Use static assert, so incorrect code isn't even compiled!
  * For instance, two separate `Type`s. One is used for registration, which iterates `Type` as well. This doesn't work in the case that the user registers different types for multiple `Wren` instances.
* There needs to be better error handling for not finding a method.
  * Is Wren actually responsible for crashing the program when a method is not found?
* Does Wren actually crash the program when an invalid operator is used on a class instance?
