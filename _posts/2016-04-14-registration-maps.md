---
layout: post
title: Registration Maps in C++
permalink: /posts/c++-registration-maps.html
---

Object maps is a construct which hits initialization order issues very
often.For example, if module is a COM server it must implement the
single entry point which must be able to create and return objects by
their identifiers. Experienced developer who is familiar with design
pattern knows pattern "Factory".

This article continues [C++
Initialization](/posts/c++-static-initialization.html).

## Straightforward Solution

The most often used solution is to create a function which knows about
all objects by including headers of the objects to function's
translation unit

```c++
// In the "Base.hpp" file
class Base
{
public:
  virtual ~Base();

  virtual void foo() = 0;
};

// In the "Object1.hpp" file
class Object1:
  public Base
{
public:
  void foo() override;
};

// In the "Object2.hpp" file
class Object2:
    public Base
{
public:
  void foo() override;
};

// In the "common-registration.cpp" file
#include "Object1.hpp"
#include "Object2.hpp"

...

Base * createObject(const char * name)
{
  if (name == nullptr) {
    return nullptr;
  }

  if (strcmp(name, "Object1") == 0) {
    return new Object1;
  } else if (strcmp(name, "Object2") == 0) {
    return new Object2;
  }

  return nullptr;
}
```

This solution works well when all folowing requirements are satisfied:

* Number of objects is relatively small. In the reality this limit
  depends on complexity of the object definitions.
  
* Object definitions are simple.

* Object's header dependencies are small and simple.

Any unsatisfied requirement from the list about will slow down
compilation when any header of object definition or its dependency is
changed.

The example above can be improved in many ways:

* using smart pointers;

* explicit memory management to avoid hitting an issue with different
  implementations of free store in different modules.

But all these issues are skipped now for clarity.

## Resolving "Slow Compilation" Issue

It is possible to avoid inclusion of all object's definition to
translation unit with factory by introducing a factory function for
every object. These factory functions should be a free function with
external linkage. Making them static member or function with internal
linkage will prevent their forward declaration and lead to including
their definitions to translation unit for factory.


```c++
// In the "Base.hpp" file
class Base
{
public:
  virtual ~Base();

  virtual void foo() = 0;
};

// In the "Object1.hpp" file
class Object1:
  public Base
{
public:
  void foo() override;
};

// In the "Object1.cpp" file
Base * Object1_create()
{
  return new Object1;
}

// In the "Object2.hpp" file
class Object2:
    public Base
{
public:
  void foo() override;
};

// In the "Object2.cpp" file
Base * Object2_create()
{
  return new Object2;
}

// In the "common-registration.cpp" file
Base * Object1_create();
Base * Object2_create();

Base * createObject(const char * name)
{
  if (name == nullptr) {
    return nullptr;
  }

  if (strcmp(name, "Object1") == 0) {
    return Object1_create();
  } else if (strcmp(name, "Object2") == 0) {
    return Object2_create();
  }

  return nullptr;
}
```

This solution compiles fast and doesn't carry any extra compilation
dependencies. But it has the following drawbacks:

1. As the first solution it requires manual management of the list of
   supported objects.
   
2. The name of object and object's implementation is separated, this
   information lives in different translation units.

3. Names of factory functions have external linkage so they should be
   selected carefully to avoid collisions.
   
To improve solution the automatic building of this registration map
can be invented.

## Avoiding Manual Management

The static or singleton collection is created to register all
supported objects. This map can be easily forward-declared and used
from the different translation units.

```c++
// In the "constructor-registration.cpp" file
typedef Base * (*factory_t)();
typedef std::map< std::string, factory_t > factory_map_t;

factory_map_t & getFactoryMap()
{
  static factory_map_t factory;

  return factory;
}

Base * createObject(const char * name)
{
  if (name == nullptr) {
    return nullptr;
  }

  factory_map_t::const_iterator item = getFactoryMap().find(name);

  return (item != getFactoryMap().end())
    ? item->second()
    : nullptr;
}

// In the "Object1.cpp" file
static Base * Object1_create()
{
  return new Object1;
}

static struct Object1Registrar
{
  Object1Registrar()
  {
    Base::getFactoryMap().emplace("Object1", &Object1_create);
  }
} object1_registrar;

// In the "Object2.cpp" file
static Base * Object2_create()
{
	return new Object2;
}

static struct Object2Registrar
{
	Object2Registrar()
	{
		Base::getFactoryMap().emplace("Object2", &Object2_create);
	}
} object2_registrar;
```

But now the factory method starts to suffer from the order of the
initialization.

All solutions for singleton can't work here because they require
There is no perfect solution for this issue. The most effective
solutions are platform dependent.

## A "Portable Solution"

Any portable solution should work within C++ standard. A possible
solution might look like:

1. All classes that must be registered inside a map are marked by a
   special macro, for example, `REGISTER_CLASS`.
   
2. The macro definition should expand to the empty line because it's
    only class mark.
    
3. A special script takes all these marks and produce an another
   translation unit which contains a generated function to perform
   operations on the classes. For COM server it can be
   `DllGetClassObject`.


In the straightforward implementation this generated translation unit
will suffer from the compilation slowdown issue. It's very important
to have simple forward'declared entity inside every class that is used
inside generated translation unit. This reduces list of the usable
constructs to the free functions. They can be generated by the macro,
they can be forward-declared inside the generated code without
including the class headers.

The single issue that is still here is the regeneration of the sources
of this translation unit after any and every change of the source
files for classes. It can be avoided by placing an additional
restriction on the source file names, for example, the registration
macro can be placed into files which names started from "`object-`".

```c++
// In the "Base.hpp" file
typedef Base * (*factory_t)();

struct object_entry_t
{
	const char * const name;
	factory_t factory;
};

#define REGISTER_OBJECT(object)

// In the "Object1.cpp" file
REGISTER_OBJECT(Object1);

// In the "Object2.cpp" file
REGISTER_OBJECT(Object2);

// In the "generated-registration.cpp" file
extern const object_entry_t * const OBJECTS_BEGIN;
extern const object_entry_t * const OBJECTS_END;

Base * createObject(const char * name)
{
  if (name == nullptr) {
    return nullptr;
  }

  for (const object_entry_t * item = OBJECTS_BEGIN; item != OBJECTS_END; ++item) {
    if (strcmp(name, item->name) == 0) {
      return item->factory();
    }
  }

  return nullptr;
}
```

The following generation script can be used to produce translation
unit with registration map.

```bash
#!/bin/sh
#
if test $(uname) = "Darwin" ; then
    sed_extended=-E
else
    sed_extended=-r
fi

tmp=$(mktemp mapXXXXXXXXXXXXXXXXXX)
# Extract unique list of the registered classes and save it to a temporary file
sed $sed_extended -ne '/#[[:space:]]*define[[:space:]]/ '\
'!s/REGISTER_OBJECT[[:space:]]*\([[:space:]]*([[:alpha:]][[:alnum:]:_]*)'\
'[[:space:]]*\)[[:space:]]*;?/\1 /gp' "$@" | sort | uniq > "$tmp"

# Generate file header
echo '#include "Base.hpp"'
# Create forward declarations
sed $sed_extended -ne 's/[^[:space:]]+/Base * &_create();/gp' < "$tmp";
# Create array declaration
echo 'const object_entry_t OBJECTS[] = {'
# Add factory functions with names
sed $sed_extended -ne 's/[^[:space:]]+/{ "&", \&_create },/gp' < "$tmp"
echo '};'
# Generate iterator range
echo 'extern const object_entry_t * const OBJECTS_BEGIN = OBJECTS;'
echo 'extern const object_entry_t * const OBJECTS_END = OBJECTS + sizeof(OBJECTS) / sizeof(OBJECTS[0]);'

rm "$tmp"
```

Even this simple script starts to suffer from platform differences:
the Darwin platform (MacOS) has different parameter for extended
regexes in "sed".

All these stuff can be integrated in make file in the following way:

```
ObjectMap.o: Object1.cpp Object2.cpp Base.hpp
        sh ./generate-map.sh $(filter-out %.hpp,$^) | $(CXX) -x c++ -c -o $@ $(CPPFLAGS) $(filter-out -M%,$(CXXFLAGS)) -
```

## Non-portable Solution

Although the portable solution is much faster and efficient than the
first one, it's safer than the version with the singleton map, it
isn't the most effective.

The most effective variant can be created if it's possible to convince
the toolchain to do this work for the developer. This will use some
kind of the platform-specific things.

The idea behind is very simple. List with the information about
objects to register must be collected by the compiler/linker not by
the source code. Both platforms offer efficient solution for this task
--- named sections in binaries. For details about sections please take
a look at the insert. This solution can be implemented by GCC on
Linux, Apple Clang on MacOS  and Microsoft Visual C++ on Microsoft
Windows.

> #### Binary Sections
>
> When a linker constructs a final binary file it places objects to
> the separate parts of the target file. These parts have  names
> and usually contain the same kind of data, for example:
>
> `.text`
> : contains all executable code;
>
> `.data`
> : contains all global initialized variables;
>
> `*.bss`
> : contains all global uninitialized variables.
>
> Developer can define its own sections and place them to the
> binary. Different toolchains offer different tools for this task
> but in general they work similar: all objects from the different
> translation units which are marked by the section name are
> placed to that section.


What can be collected to the additional section?

The collected data should be in a simple format which can be easily
handled by generic code. Executable code can't meet this requirement
because of different function sizes. The simplest format of the
section is a list of function pointers. These pointers can be a global
variables initialized automatically during module load and these
variables are marked with unique section name. The linker will collect
all these variables to the section section building an array. Because
size of these pointers is fixed it can be easily iterated if size of
the section is known.

Actual compiler and linker behaviors are a little bit different. The
following code shows how to perform this task with GCC. To show a more
general approach sample creates array of structures with object names
and factory methods as the sample above.

```c++
// In the "Object1.cpp" file
extern const object_entry_t object1_registrar __attribute__((section("OBJECTS"))) =
{
  "Object1",
  &Object1_create
};

// In the "Object2.cpp" file
extern const object_entry_t object2_registrar __attribute__((section("OBJECTS"))) =
{
  "Object2",
  &Object2_create
};

// In the "section-registration.cpp" file
extern object_entry_t OBJECTS_BEGIN;
extern object_entry_t OBJECTS_END;

Base * createObject(const char * name)
{
  if (name == nullptr) {
    return nullptr;
  }

  for (object_entry_t * item = &OBJECTS_BEGIN; item != &OBJECTS_END; ++item) {
    if (strcmp(name, item->name) == 0) {
      return item->factory();
    }
  }

  return nullptr;
}
```

To properly generate variable `OBJECTS_BEGIN` and `OBJECTS_END` the
following script should be passed to LD linker:

```
SECTIONS
{
  OBJECTS :
  {
    OBJECTS_BEGIN = . ;
    *(OBJECTS) ;
    OBJECTS_END = . ;
  }
}
INSERT AFTER .rodata ;
```

The makefile update:

```
section-regstration: section-registration.o Base.o Object1.o Object2.o objects.ld
        g++ -Wall -o $@ $(filter-out objects.ld,$^) -Tobjects.ld
```

Although GCC exist on Microsoft Windows platform I can't convince
toolchain to produce proper PE binary with this trick.

Also, Apple Mac OS X platform uses Clang for the toolchain which looks
like GCC-compatible toolchain. But the linker for MachO binaries can't
accept linker scripts and doesn't sort symbols in the sections so this
trick doesn't work on the Apple's platform too. But Apple's toolchain
offers another way to make such registration map:

```c++
// In the "Object1.cpp" file
extern const object_entry_t object1_registrar __attribute__((used,section("__DATA,__OBJECTS"))) =
{
  "Object1",
  &Object1_create
};

// In the "Object2.cpp" file
extern const object_entry_t object2_registrar __attribute__((used,section("__DATA,__OBJECTS"))) =
{
  "Object2",
  &Object2_create
};

// In the "section-registration.cpp" file
extern object_entry_t OBJECTS_BEGIN __asm("section$start$__DATA$__OBJECTS");
extern object_entry_t OBJECTS_END  __asm("section$end$__DATA$__OBJECTS");

Base * createObject(const char * name)
{
  if (name == nullptr) {
    return nullptr;
  }

  for (object_entry_t * item = &OBJECTS_BEGIN; item != &OBJECTS_END; ++item) {
    if (strcmp(name, item->name) == 0) {
      return item->factory();
    }
  }

  return nullptr;
}
```

Microsoft Visual C++ can do a similar trick. For example, ATL since
version 7.0 uses it to make `DllGetClassObject` implementation more
effective at compile time. But it's really tricky and relies on some
semi-documented compiler and linker features. So it would be wise to
avoid its usage.


## Conclusion

The order of the global object initialization can give a lot of issues
during implementation of registration maps. It's much safer to use POD
types as global objects because they initialized before any code
inside module is executed. But usage of such types as registration
maps increases manual work or produces platform-dependent solutions.
