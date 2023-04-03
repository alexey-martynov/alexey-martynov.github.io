---
layout: post
title: About Static Initialization in C++
permalink: /posts/c++-static-initialization.html
---

Initialization of a C++ program might be very tricky due to amount of
facilities involved in the process. The possible pitfall and solutions
discussed here.

* toc
{:toc}

## A Sad Story

One day Bob, a very skilled C++ developer, discovered a probem: he
wanted to have a single instance of an object in his program. We will
name this object as `Facility` during this story.

His experience said:
  
---&nbsp;Your implementation of the `Facility` is very complex. You are
using a lot of external stuff, a lot of helpers which are specific to
the `Facility`. Why don't try to hide them?

--- OK, --- Bob thought. --- I know a good pattern for it --- the
"Singleton". I'll use it along with PIMPL idiom. The details of the
`Facility` implementation will be hidden and I will have the single
instance of the `Facility` in my program.

With this idea Bob wrote the following code:

```c++
// In the "Facility.hpp" file
class Facility {
public:
  static Facility &amp; getInstance();

  virtual void foo();
  …

protected:
  Facility();
  ~Facility();

private:
  Facility(const Facility &) = delete;
  Facility(Facility &&) = delete;
  Facility & operator =(const Facility &) = delete;
  Facility & operator =(Facility &&) = delete;
};

// In the "Facility.cpp"
#include "Facility.hpp"

#include <memory>

#include <boost/thread/once.hpp>

class FacilityImpl:
    public Facility {
public:
  FacilityImpl();
  ~FacilityImpl();

  void foo() override;
};

static std::unique_ptr<FacilityImpl> instance;
static boost::once instance_once = BOOST_ONCE_INIT;

static void createFacility()
{
  instance.reset(new FacilityImpl);
}

Facility & Facility::getInstance()
{
  boost::call_once(&createFacility, instance_once);

  return *instance;
}

…

// Somewhere on the other file
class Object {
public:
  Object()
  {
    Facility.getInstance().foo();
  }
};

static Object object;
```

Note: for C++ 03 the `std::auto_ptr` or `boost::scoped_ptr` should be used.

The code was successfully compiled and passed all tests, it was looked
perfect:

* The details of the implementation are hidden.

* Clients are unable to destroy object by taking address
  from the reference and calling `delete`.

* Nobody can make another instance of the `Facility` by copying.

* Nobody can hijack instance of the `Facility` by moving.

* `Facility` is available during startup.

He committed it to the version control system.

Two days later Bob had realized that the code has started to crash. He
spent a lot of time trying to understand what is happening. Finally,
he found the issue --- the constructor of the `std::unique_ptr` for
the variable `instance` is called *after* the first call to the `Facility::getInstance()`.

Now he needs to fix this implementation.

## Program Startup

The machinery behind Bob's adventures is named "dynamic object of an
object in namespace scope". It's important part of any C++ program.

At the beginning the rules are pretty easy:

> The order of the initialization of the global objects
  is undefined between the translation units.

The full details of this rule can be found in the C++ Standard (for
example, ISO IEC 14882-2003, section 3.6.2 paragraph 3, or, with newer
standard, ISO IEC 14882-2020, section 6.9.3.3).

In the Bob's story this means that nobody can say what will happen
earlier: construction of the `instance` or construction of the
`object`. But everybody can ask why it worked perfectly when Bob
tested the code? "Undefined" means "undefined" literally. It isn't
wise to rely on it because it can change at any time under any
external circumstance which is hard to control For example, possible
aspects that control the order of initialization might be:

* An order in which individual translation units are passed to the
  linker.
  
* Compile- and link-time optimizations.

* Versions of the compiler and linker.

It's very surprising how easy to get into this pitfall. Even for the
best developers. Especially when all things become complicated and a
deadline comes closer.


But C++ standard misses the one important part. When we are trying to
apply it to the specific platform --- Microsoft Windows or Linux ---
the plaform specific things come into play. For example, Shared
Objects (on Linux) or Dynamic Link Libraries (on Microsoft
Windows). These facilities work very tightly with their platform and
perform very complex tasks to resolve dependencies, addresses of
different entities, initialization/deinitialization and so on.

The unavoidable part of these facilities is backward
compatibility. For the sake of backward compatibility very terrible limitations can be introduced.

Microsoft Corporation has a very good document named "DLL Best
Practices". It contains all required details how to properly write DLL
code. This document must be read by all developers who write in C and
C++ for the Microsoft Windows platform.

The ELF and Linux impose less restrictions but it's preferrable to use
strict rules from Microsoft because it allows future porting of the
program to the Microsoft Windows platform.

## Solution

It's very sad but no perfect solution exists for the Bob's
problem. Lets take a look at the possible solutions and their
drawbacks.

### Remove `std::unique_ptr`

The code will look like:

```diff
- static std::auto_ptr<FacilityImpl> instance;
+ static FacilityImpl * instance = nullptr;
  static boost::once instance_once = BOOST_ONCE_INIT;

  static void createFacility()
  {
-   instance.reset(new FacilityImpl);
+   instance = new FacilityImpl;
  }
```

This solution has obvious drawback: the `FacilityImpl` instance will
definitely leak. It might be not dangerous in some cases but when
`FacilityImpl` lives in shared object and contains an execution thread
it's very important to prevent the thread survive after library
unloaded. It is possible to prevent this by calling `atexit` or
another appropriate function to schedule object destruction on
exit. But this leads us to another issue: the instance might die
__before__ its last client depending on the implementation. It might
not be a problem again but this case be must considered before
implementing.

### Remove Boost and Use C++ 11 or Newer Standard

The C++ 11 introduced multithreading support to C++ standard. This
means that standard library and *language* knows about threads and
properly handles them. Being applied to this task it gives thread safe
initialization so it is possible to remove `boost::once` and just
return reference to local static variable:

```diff
- static std::unique_ptr<FacilityImpl> instance;
- static boost::once instance_once = BOOST_ONCE_INIT;
- 
- static void createFacility()
- {
-   instance.reset(new FacilityImpl);
- }

  Facility & Facility::getInstance()
  {
-   boost::call_once(&createFacility, instance_once);
+   static FacilityImpl instance;

-   return *instance;
+   return instance;
  }
```

Unfortunately this solution is slow because every time when
`Facility::getIstance()` is called the following steps performed:

1. Take lock on local static variable `instance`.

2. Check that it is already constructed.

3. If not try to construct it.

4. In case of exception during construction release lock and propagate
   exception to the caller.
   
5. Mark object as created.

6. Release lock.

So this is not an option for performance critical code.

In case of C++ 03 it is looks like thread unsafe construction but at
least GCC ensures thread safety here.

More important that this solution has the same issue as previous one with
`atexit` --- `instance` might die before its last client.

### Create Explicit Initialization and Deinitialization

```diff
- static std::unique_ptr<FacilityImpl> instance;
- static boost::once instance_once = BOOST_ONCE_INIT;
- 
- static void createFacility()
- {
- 	instance.reset(new FacilityImpl);
- }
+ static FacilityImpl * instance = nullptr;
+ static init init_count = 0;
 
  Facility & Facility::getInstance()
  {
-   boost::call_once(&createFacility, instance_once);
+   if (instance == nullptr) {
+     throw std::logic_error("System is not initialized");
+   }

    return *instance;
  }
+
+ void initialize()
+ {
+   if (init_count == 0) {
+     assert(instance == nullptr);
+     instance = new FacilityImpl;
+   }
+
+   ++init_count;
+ }
+
+ void deinitialize()
+ {
+   assert(init_count != 0);
+   if (--init_count == 0) {
+     delete instance;
+     instance = 0;
+   }
+ }
```

> Thread safety support has been omitted in the
  initialization/deinitialization code for the sake of clarity.

Apparently, the requirement to call `initialize()` and
`deinitialize()` might be troublesome for clients. At least it's
important to not get into another pitfall --- allow one single call to
`initialize()`. In a system with a lot of independent components it's
very hard to control such initialization. So it's better to allow
multiple calls to the initialization function. In the worst case
client can call it before every operation with corresponding component.
The [Schwartz/Nifty
Counter](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Nifty_Counter)
can assist in such initialization and deinitialization. 

## Conclusion

The is no completely safe way to get guaranteed order of
initialization of objects in namespace scope. The different solutions
have different drawbacks so every particular case needs careful
investigation and selected solution should be documented.

In general the [Dependency
Injection](https://en.wikipedia.org/wiki/Dependency_injection) solves
all issues with dependencies. But it might be tedious to provide
global dependencies like logger via Dependency Injection.

In the [C++ Registration Maps](/posts/c++-registration-maps.html) I'll
continue to explore possible ways to make safe namespace scope object
initialization.
