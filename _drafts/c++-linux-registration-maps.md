---
layout: post
title: Registration Maps and Shared Objects
permalink: /posts/c++-linux-registration-maps.html
tag: C++
---

{% assign previous_post = site.posts | where: "url", "/posts/c++-registration-maps.html" | first %}
The technique from a [{{previous_post.title}}]({{previous_post.url}})
post works fine when program is linked statically. But most of
programs is linked dynamically and this breaks logic from previous [article]({{previous_post.url}}).

This article continues
[{{previous_post.title}}]({{previous_post.url}}).

## Table of Contents
{:.no_toc}
* toc
{:toc}

## Static vs. Dynamic Linking

Static linking leads to some interesting aspects:

* Since all reusable parts of program are linked statically this
  dramatically increases builds because every single binary, for
  example, main program and unit tests contain the same code.

* This makes impossible to make small updates by replacing dynamic
  libraries.

* There is no plugin system and program is not extensible. Lets omit
  discussion about dynamic libraries plugins versus plugins in
  separate processes since this is not a topic of current post.

On other side not using dynamic libraries gives some
benefits:

* Additional work to maintain binary compatibility which is very
  complex task for C++ programs is not required because everything is
  integrated to single binary.

* Dependency tracking is finished during the build process. No
  "missing library" and similar issues.

When program is linked with dynamic libraries which should contribute
to existing registration maps the task becomes complex because of way
how dynamic libraries work on Linux platform.

## Dynamic Libraries on Linux system

The detailed description of dynamic library behavior on Linux system
should be found in `LD` documentation.

### Static Library vs Shared Object

The static libraries are archives of object files with additional index
to speed up symbol search. The `ar` program is used to create them
from individual files. The static libraries don't carry any
information about their dependencies so client should manually add
them to linker command line to have linking finished successfully.

The process of linking has one important caveat: the `LD` as linker is
a single pass linker. This means that it pass over the libraries once
and when it processes any library it drops all unreferenced symbols
from it. So if next library requires symbol from current library it
will not find it and linking is failed. Solution: sort libraries from
most specific to generic, for example, the standard C library should
be the last and the standard C++ library should be just before it.

The dynamic library named "Shared Object" on Linux platform is a
partially linked program. All object files are linked together and
nothing is dropped. In general all unresolved symbols should be
available at the time of shared object linking either from another
shared object or from static library. The reference to such shared
objects will be added to header and this libraries will be
automatically loaded.

If dependency is a static library it will be linked to resulting
shared library: all object files with referenced symbols will be added
to shared object and all other objects files are dropped. As the
result:

> Static library linked to Shared Object (partially) contribute to
> list of exported symbols for target Shared Object.

So if any static library is linked to many shared objects it is
possible to have duplicate symbol definitions during final link of
program.

### Symbol Search

On Linux platform all loaded into process libraries form a tree. To
call any function or take value of variable its name (symbol) should
be found. Without special steps the symbol searched by its name it a
tree using breadth first order.

If more that library provides the same symbol the symbol from the very
first shared object (breadth first) will be used by _all_ loaded
shared objects including the main program. This makes safe? for
example, passing pointers from `malloc()` between shared objects and
`free()` them from another shared object.

Please note that Windows platform works in another way and all symbols
from dynamic libraries are reference not only by its name but a
library name too.

The important aspect of this search that at runtime there no
"duplicate symbols" issues. Symbol either exists in process or
missing. If symbol is missing the program fails.

#### Weak and Strong Symbols

By default all symbols are strong. This means that during linking all
they references should be resolved to static or shared library. At
runtime this symbols are always available and not null.

The ELF format allows to specify additional attribute `weak` for every
symbol to change behavior (use `__attribute__((weak))` in GCC):

* The weak symbols are not required. For example, declaring function
  as weak makes possible to have empty (`nullptr`) address of it
  because it is missing in resulting binary.

  This technique is used to create hooks for library in client code so
  do not foget to check that it is not `nullptr` before call.

* The weak symbols may be encountered in symbol search during linking
  multiple times even with strong symbol. When strong symbol
  encountered twice the error "duplicate symbol" issued. When multiple
  weak symbols encountered the first one is used. In case of multiple
  weak symbols and single strong symbol the strong one wins and used
  as the result.

  This technique is used to provide default implementations which are
  easy to override in client code.

* During runtime the same symbol selection algorithm may be used but
  on modern systems the `LD_DYNAMIC_WEAK=1` might set to environment
  to have non-first strong symbol to win.

  In addition to have the main binary symbol win the `-rdynamic`
  parameter required when linking main binary. This will export
  symbols from it as for shared object and they will participate in
  symbol search and selection.

> The implicit template instantiating process always creates weak
> symbols. This allows to merge same symbols from different object
> files to a single one. But this process might be broken for `static`
> methods of template classes. These methods will have internal
> linkage so the can't be merged.

#### Overriding Symbols At Runtime

The process of symbol search described above allows interesting
technique of symbol interception. The variable `LD_PRELOAD` main
contain names or paths of shared object to load first _before_
program. Since they loaded before program they symbols win in symbol
selection. This allows to substitute implementations.

Please take into consideration the following aspects:

1. In general, all set of functions should be intercepted
   together. For example, if `malloc()` is intercepted the following
   function form _incomplete_ list should be intercepted too:

   * `calloc()`
   * `free()`
   * `realloc()`
   * `reallocarray()`
   * `posix_memalign()`

2. When interceptor should use default implementation it should be
   found by calling `dlsym()` using `RTLD_NEXT` special
   handle. Please look to `dlsym()` documentation for details.

#### Symbol Visibility

The C and C++ languages knows nothing about dynamic libraries. They
have concepts of "internal" and "external" linkage and nothing
more. The symbols with internal linkage are accessible inside the same
translation unit and symbols with external linkage are accessible
everywhere.

The C++ offers anonymous namespaces: symbols from anonymous namespace
may have external linkage but they always has unique name for every
translation unit so they don't clash. This requires because classes
can't have internal linkage.

> NOTE: The free functions from anonymous namespaces receive internal
> linkage implicitly.

> NOTE: The GDB might have issues during lookup of symbols (variables
> or functions) in anonymous namespaces when current context is not
> corresponding translation unit. For such cases use `static` instead
> of anonymous namespace.

When shared objects come into play it contribute additional concept:
visibility.

The interesting variants of visibility is `default` and `hidden`.

The `default` visibility means that symbol is visible to other shared
objects. This is similar to external linkage.

The `hidden` visibility prevents accessing symbol from outside shared
objects. This is similar to internal linkage at the dynamic library
level.

The visibility should be taken in consideration when creating library
or framework code. Since default visibility for all symbols can be
defined by compiler parameters the symbols which are entry points of
library should be marked as `__attribute__((visibility("default")))`
explicitly.

This allows to hide implementation details inside the shared object:

* The default visibility is set to `hidden` via `-fvisibility=hidden`
  effectively hiding everything in shared object.

* The public (client) interface is explicitly marked with
  `__attribute__((visibility("default")))`.

> NOTE: The C++ exceptions require additional attention: they require
> type info lookup so don't forget to make them visible outside.

### Loading, Unloading and Using Symbols at Runtime

The usage of shared objects is not limited to startup of process. It
is possible to load and unload shared object at runtime. To achieve
this the following functions from `libdl` are used:

* `dlopen()` loads shared object by name or path and returns "handle"
  to it.

* `dlclose()` unloads shared object. If this is the last reference to
  shared object it will be unloaded from memory invalidating all
  references and pointers to it.

* `dlsym()` finds symbol by name using handle to shared object. Some
  special predefined handles are available like `RTLD_NEXT`.

  Please note that `dlsym()` performs breadth first search in tree of
  shared objects so it is _possible_ to have valid pointer to symbol
  which is not provided from shared object from the first
  parameter. In such case symbol will be provided from the first
  dependency of specified shared object. This is no obvious and
  contradicts Windows platform behavior.

## Straightforward Processwide Registration Map

One of the possible ways to construct processwide registry map is to
register current shared object's registration section in central
registry.

For example, if all shared objects with registration map provide weak
function `void registerModule(const module_info_t*)` then any shared object
can call it providing information about itself. The module info can
look like (using [previous article](previous_post.url) sample):

```c++
struct module_info_t {
  object_entry_t * begin;
  object_entry_t * end;
};
```

It can be filled at any translation unit within shared object like:

```c++
const object_entry_t* __start_OBJECTS;
const object_entry_t* __stop_OBJECTS;

const module_info_t module_info {
    .begin = __start_OBJECTS,
    .end = __stop_OBJECTS,
};
```

As the result the main trick to have a call to
`registerModule()`. According to rules of symbol selection this will
be the very first symbol in process regardless of amount of this
symbols in shared objects. If access to registered modules information
i placed to the same translation unit the neighbor function will
access shared data in registered modules list.

The possible ways to have this call done upon module startup:

* Create a class then create an instance of the class as global
  variable and call `registerModule()` in constructor:

  ```c++
  class ModuleRegistrar {
  public:
    ModuleRegistrar()
    {
      registerModule(&module_info);
    }
  } module_registrar;
  ```

* Use constructor functions from shared object.

  The linker allows to add attribute `constructor` (and `destructor`
  too) to any free function without parameters and without
  result. This function will be called when shared object is
  loaded. For example:

  ```c++
  __attribute__((constructor))
  static void initModule()
  {
    registerModule(&module_info);
  }
  ```


It would be wise to add attribute `used` to `module_registrar` and
`initModule()` to avoid accidental removal of unreferenced symbols.

Both solutions suffer from order of initialization issue: compiler,
linker and runtime linker do not provide any guarantee of order of
calls to constructors. As the result construction of other global
objects might not find module because it is not registered yet.

## Enumerating Shared Objects

To avoid issue with order of initialization another solution
required. The `dl_iterate_phdr()` can assist to solve the issue.

This function allows to list all loaded shared objects including the
main program. So it is possible to do the following steps to get
complete registration map:

1. Iterate shared objects.
2. For every shared object get handle to it.
3. Look for specific symbol describing module in every shared object.
4. If symbol is found take data and build registration map.

This looks like a simple task but some caveats may break the idea:

* The first issue is that `dl_iterate_phdr()` gives information about
  file name of loaded shared object in `dlpi_name` member but not a
  handle. And this name will be empty for the main program.

* The handle can be obtained via `dlopen()`. The first parameter of
  `dlopen()` is a name or path of shared object to load. The `NULL`
  (`nullptr`) will return handle for the main program.

* The returned handle should be released via `dlcose()` call.

* The symbol lookup is performed via `dlsym()` function. Since it
  performs breadth fist search over dependency tree the received
  non-`NULL` addresses should be deduplicated, for example, placing
  them to a `std::set`. This makes impossible to tell which exact
  shared object contains symbol but usually this is not an issue.

* The C++ symbols are mangled so `initModule` becomes something
  special. The demangled names can be viewed with `nm -C <so file>`
  utility. The `dlsym()` function knows nothing about mangling and
  looks for exact name of symbol.

  To mitigate this problem the symbol should have `extern "C"`
  declaration. This is not a problem for function but for global
  variables the `extern "C"` block should be used. For example:

  ```c++
  extern "C" {
    const module_info_t module_info {
      .begin = __start_OBJECTS,
      .end = __stop_OBJECTS,
    };
  }
  ```

* There is no simple way to detect whether pointer from `dlsym()`
  points to function or variable.

Taking into account everything above the following function will
iterate over all registration entries in running process:

```c++
using module_list_t = std::set<const module_info_t *>

…

static int on_header(dl_phdr_info * info, size_t size, void * data)
{
    module_list_t * ctx = static_cast<module_list_t *>(data);
    void * handle = dlopen((((info->dlpi_name == nullptr) || (*info->dlpi_name == 0x00))
          ? nullptr
          : info->dlpi_name),
        RTLD_LAZY);

    if (handle != nullptr) {
        void * module_addr = dlsym(handle, "module_info");

        dlclose(handle);

        if (module_addr != nullptr) {
            modules->insert(static_cast<const module_info_t *>(module_addr));
        }
    }

    return 0;
}

void populate()
{
  module_list_t modules;

  dl_iterate_phdr(on_header, &modules);
}
```

The test for main program shared object is a little bit paranoid
because it handles empty string and missing string but shared object
enumeration shouldn't be on critical path so playing safe is better.

The logic can be converted to a C++ iterator. This allows simple
integration with other C++ facilities.

## Unloading Shared Object

The loaded shared objects fall to one of the following categories:

* Can't be unloaded. This is a main program and all its dependencies
  including transitive. Everything loaded via `LD_PRELOAD` placed to
  this category too.

* Can be unloaded. All other shared objects loaded via `dlopen()` in
  this category.

Unloading shared object is a very dangerous operation since it
invalidates all pointers and references contained in unloaded shared
object and it dependencies too. This includes:

* Pointers to functions and variables in shared object.
* C++ type info from shared object.
* Virtual Tables for types from shared object.

So every unload should be performed with extra care. In multithreaded
program this task becomes more complex because one thread might try to
unload shared object while other iterates over it.

The best way to solve the issue is pinning of the shared object: every
pointer to dynamically loaded shared object is bound with handle to
that shared object. When all pointers to shared object released the
shared object can be unloaded.

To achieve this a pair of classes can be created:

* `DynamicModule` represents loaded shared object and holds handle.
* `DynamicModuleSymbol` represents resolved symbol and holds pointer
  to symbol _and_ handle.

Although it is possible to increase shared object reference counter by
calling `dlopen()` again it is inefficient because:

1. `dlopen()` not so fast because requires search.
2. Requires to hold name of the shared object which is a string. In
   general case it is a full path of the module.

So it is more efficient to create a structure with 2 members: handle
and reference count. Pointer to this structure will be shared between
`DynamicModule` and `DynamicModuleSymbol`.

The `DynamicModuleSymbol` is a template receiving a type of value to
point to. The `DynamicModule` resolves symbol to pointer, casts it to
specific type and returns `DynamicModuleSymbol`.

So implementation of `DynamicModule` can look like this:

```c++
class DynamicModule
{
public:
  DynamicModule(const char* name, int flags = RTLD_LAZY):
      handle_{new module_handle_t}
  {
    handle_->refs = 1;
    handle_->handle = dlopen(((name != nullptr) && (*name != 0x00)) ? name : nullptr, flags);

    if (handle_->handle == nullptr) {
      delete handle_;

      throw std::runtime_error{dlerror()};
    }
  }

  DynamicModule(const std::string& name):
      DynamicModule(name.c_str())
  {
  }

  DynamicModule(const DynamicModule& rhs) noexcept:
      handle_{rhs.handle_}
  {
    if (handle_ != nullptr) {
      ++handle_->refs;
    }
  }

  DynamicModule(DynamicModule&& rhs) noexcept:
      handle_{std::exchange(rhs.handle_, nullptr)}
  {
  }

  ~DynamicModule() noexcept
  {
    reset();
  }

  DynamicModule& operator=(const DynamicModule& rhs) noexcept
  {
    if (this != &rhs) {
      reset();

      handle_ = rhs.handle_;
      if (handle_ != nullptr) {
          ++handle_->refs;
      }
    }

    return *this;
  }
  DynamicModule& operator=(DynamicModule&& rhs) noexcept
  {
    if (this != &rhs) {
      reset();

      handle_ = std::exchange(rhs.handle_, nullptr);
    }

    return *this;
  }

  explicit operator bool() const noexcept
  {
    return handle_ != nullptr;
  }

  bool operator!() const noexcept
  {
    return handle_ == nullptr;
  }

  bool operator==(const DynamicModule& rhs) const noexcept
  {
    return (handle_ == rhs.handle_)
        || ((handle_ != nullptr)
            && (rhs.handle_ != nullptr)
            && (handle_->handle == rhs.handle_->handle));
  }

  bool operator!=(const DynamicModule& rhs) const noexcept
  {
    return !operator==(rhs);
  }

  template <typename T>
  DynamicModuleSymbol<T> get(const char* name)
  {
    if (handle_ == nullptr) {
      throw std::runtime_error{"Lookup for symbol without shared object is forbidden"};
    }

    return DynamicModuleSymbol<T>{handle_, static_cast<T*>(dlsym(handle_->handle, name))};
  }

  template <typename T>
  DynamicModuleSymbol<T> get(const std::string& name)
  {
    return get<T>(name.c_str());
  }

  void reset() noexcept
  {
    if (handle_ != nullptr) {
      if (--handle_->refs == 0) {
        delete handle_;
      }
      handle_ = nullptr;
    }
  }

private:
  struct module_handle_t
  {
    void* handle;
    std::atomic<int> refs;
  };

  module_handle_t* handle_;

  template <typename>
  friend class DynamicModuleSymbol;
};
```

The implementation of `DynamicSymbolModule` is a little bit tricky
because it s convenient to behave as smart pointer when type of symbol
is value and behave as function object when type of symbol is
function. To achieve this the C++ Concepts can be used as requirement
for members:

* The `operator *` and `operator ->` should be available when type is
  not a function.
* The `operator ()` should be available when type is function.

As the result might be implemented as:

```c++
template <typename T>
class DynamicModuleSymbol
{
public:
  DynamicModuleSymbol(const DynamicModuleSymbol<T>& rhs) noexcept:
      handle_{rhs.handle_},
      ptr_{rhs.ptr_}
  {
    if (handle_ != nullptr) {
      ++handle_->refs;
    }
  }
  DynamicModuleSymbol(DynamicModuleSymbol<T>&& rhs) noexcept:
      handle_{std::exchange(rhs.handle_, nullptr)},
      ptr_{std::exchange(rhs.ptr_, nullptr)}
  {
  }

  DynamicModuleSymbol<T>& operator=(const DynamicModuleSymbol<T>& rhs) noexcept
  {
    reset();

    handle_ = rhs.handle_;
    ptr_ = rhs.ptr_;
    if (handle_ != nullptr) {
      ++handle_->refs;
    }

    return *this;
  }

  DynamicModuleSymbol<T>& operator=(DynamicModuleSymbol<T>&& rhs) noexcept
  {
    if (this != &rhs) {
      reset();

      handle_ = std::exchange(rhs.handle_, nullptr);
      ptr_ = std::exchange(rhs.ptr_, nullptr);
    }

    return *this;
  }

  explicit operator bool() const noexcept
  {
    return ptr_ != nullptr;
  }

  bool operator!() const noexcept
  {
    return ptr_ == nullptr;
  }

  bool operator==(const DynamicModuleSymbol<T>& rhs) const noexcept
  {
    return ptr_ == rhs.ptr_;
  }

  bool operator!=(const DynamicModuleSymbol<T>& rhs) const noexcept
  {
    return ptr_ != rhs.ptr_;
  }

  T& operator*() const noexcept requires(!std::is_function_v<T>)
  {
    return *ptr_;
  }

  T* operator->() const noexcept requires(!std::is_function_v<T>)
  {
    return ptr_;
  }

  template <typename... Args>
  requires std::invocable<T, Args...>
  std::invoke_result_t<T, Args...> operator()(Args&& ... args) noexcept(std::is_nothrow_invocable_v<T, Args...>)
  {
    return (*ptr_)(std::forward<Args...>(args)...);
  }

  void reset() noexcept
  {
    if (handle_ != nullptr) {
      if (--handle_->refs) {
        delete handle_;
      }
      handle_ = nullptr;
      ptr_ = nullptr;
    }
  }

private:
  DynamicModule::module_handle_t* handle_;
  T* ptr_;

  DynamicModuleSymbol(DynamicModule::module_handle_t* handle, T* ptr) noexcept:
      handle_{handle},
      ptr_{ptr}
  {
    if (handle_ != nullptr) {
      ++handle_->refs;
    }
  }

  friend class DynamicModule;
};
```

> NOTE: The examples above depend on each other. To have them compiled
> the code should be slightly restructured. Inline implementation is
> used only to have all parts of every class in a single place to
> achieve a small piece of clarity.

If `DynamicModuleSymbol` should be placed to a container like
`std::set` the comparator is required. It is possible to write a set
of comparison operators (less, less than, greater and greater than)
but pointer to symbol in program doesn't have such semantics. So
comparator should be implemented as separate friend class.

## Conclusion

The two ways of process-wide registration map implementation have its
own pros and contras:

* The constructor-based implementation suffers from order of
  initialization issues and might hit threading issues during
  constructor call.

* The module enumeration version doesn't have issues with
  initialization but requires proper handling shared object unload to
  avoid dangling pointers.

The implementation should be selected in accordance with task
limitations.
