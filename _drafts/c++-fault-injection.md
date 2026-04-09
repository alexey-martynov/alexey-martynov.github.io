---
layout: post
title: Fault Injection in C++
permalink: /posts/c++-fault-injection.html
tag: C++
---

It is obvious that any software requires testing. Many guides shows
how to write unit, functional and integration tests. But when software
integrates many components all of these tests hit the following
problem: how to test error handling and recovering when lower layers
may fail.

The different programming languages have various methods to solve such
issues but C++ offers additional ways.

> Although this article uses C++ most of these concepts can be used in
> C. Since C lacks Object-Oriented concept additional efforts are
> required to simulate behavior from C++.

Lets start with most obvious and frequently used pattern: Dependency
Injection.

# Dependency Injection

The main idea of Dependency Injection is hiding details and loosing
coupling: components don't create and use other components. They use
interfaces instead and instances of these interfaces provided to
components from external source. For example, component which reads
and writes block volume can use the following interface:

```c++
class IVolume {
public:
    virtual void read(lba_t lba, std::span<std::byte> buffer) = 0;
    virtual void write(lba_t lba, std::span<const std::byte> buffer) = 0;

protected:
    ~IVolume() = default;
};
```

The interface `IVolume` provides two methods to read and write buffer
at specific volume offset which is provided via type `lba_t` defined
elsewhere. The destructor of such interface should be:

* non-virtual and protected to prevent interface destruction by the
  client or
* virtual and public to share ownership.

In any case components will take instances of this interface by
pointer (smart pointer) or reference.

Also this interface places some restrictions in arguments:

* `lba` should be multiple of some constant, most of systems require
  4&nbsp;KiB now but 512 bytes might be used too.
* The data size in `buffer` should be multiple of some constant. This
  constant the same as for `lba` usually.
* Depending on operation the data in `buffer` should be specifically
  aligned.

So component can take instance of such interface as constructor or
method parameter. For example, function that writes pattern might
look like:

```c++
void fillVolume(IVolume& volume, lba_t size, std::span<const std::byte> pattern)
{
    if ((size % pattern.size()) != 0) {
        throw std::invalid_argument("The size should multiple of pattern size");
    }

    for (lba_t offset{}; offset < size; offset += pattern.size()) {
        volume.write(offset, pattern);
    }
}
```

Simple as all examples expected to be. Testing of this function is
easy: provide instance of `IVolume` interface which remembers
operations and validate that all volume is covered at the end:

```c++
class TestVolume: public IVolume {
public:
    void read(lba_t, std::span<std::byte> buffer) override
    {
        throw std::logic_error("Not implemented");
    }

    void write(lba_t lba, std::span<const std::byte> buffer) override
    {
        written_.emplace(lba, buffer.size());
    }

    bool fullyWritten(lba_t size) const;

private:
    std::map<lba_t, std::size_t> written_;
};
```

The `fillVolume()` is very straightforward function so no complex
error handling inside: all errors from volume propagated as is to
caller. In case of faulty disks as all hardware do this function might
retry writes, for example:

```c++
void fillVolume(IVolume& volume, lba_t size, std::span<const std::byte> pattern,
    int retries = 3)
{
    if ((size % pattern.size()) != 0) {
        throw std::invalid_argument("The size should multiple of pattern size");
    }

    for (lba_t offset{}; offset < size; offset += pattern.size()) {
        int tries_left = retries;

        while (true) {
            try {
                volume.write(offset, pattern);
                break;
            } catch (...) {
                if (--tries_left <= 0) {
                    throw;
                }
            }
        }
    }
}
```

Since interface uses exceptions the `try` block is unavoidable. The
loop for each block is required to implement retries but the way of
handing success operations as a matter of discussion. Infinite loop
with `break` inside when write succeeds, using `tries_left` check in
condition and zeroing variable when write success: all variants
possible and depend on author's taste.

To test such function with retries the `TestVolume` should provide
errors. For example,

```c++
class TestVolumeWithErrors: public IVolume {
public:
    void read(lba_t, std::span<std::byte> buffer) override
    {
        throw std::logic_error("Not implemented");
    }

    void write(lba_t lba, std::span<const std::byte> buffer) override
    {
        if (auto error = errors_.find(lba); error != errors_.end()) {
            errors_.erase(error);
            throw std::system_error(EIO, std::system_category(),
                "INJECTED ERROR");
        }

        written_.emplace(lba, buffer.size());
    }

    bool fullyWritten(lba_t size) const;

private:
    std::map<lba_t, std::size_t> written_;
    std::multiset<lba_t> errors_;
};
```

This implementation checks that `lba` argument is in set of faulty
LBAs and report error in that case. The faulty LBAs stored in multiset
allowing to report multiple errors in the single LBA. The `written_`
member contains only successful operations.

This test volume allows to check every possible execution path in
`fillVolume` depending on how `errors_` if filled:

* Empty error list checks happy path.
* Adding errors for some LBAs checks the retries are performed.
* Placing more than `retries` errors for single LBA checks that
  failure is reported.

Creating interfaces and mocking it is easy, straightforward and almost
unavoidable in languages like Java with a great help of mocking
libraries and frameworks.

But in C++ this places some restrictions:

* The lower level should provide interface with virtual functions. In
  Java all functions are virtual but in C++ class might not have any
  virtual function to increase performance because virtual function
  call requires additional memory lookup.
* The lower level might be a system call such as `open()` or
  `write()`. Wrapping all of this stuff to interfaces slows down
  development and decreases performance.

Additional issue with dependency injection: in case of integration
testing the program should be built with test implementations
included. External API and/or configuration files should handle such
test implementations and so on. It is not a very big issue to build
such binary, for example, development process can use various
sanitizers and all of them require special build. But efforts should
be reduced.

# Handling C functions and system calls

The system calls are represented as free functions like `write()` or
`close()`. To simulate errors from such functions dependency injection
might work but code become complicated. The following ways are
available without any low level hacks:

1. Introduce abstraction like "IFile" and provide default
   implementation which perform system calls. In tests the test
   version should be provided similar to previous section. This might
   require complex Dependency Injection framework or creation of
   factories.

2. Inject pointers to functions. This way require injection of set of
   related functions, for example, `creat()`, `read()` and
   `close()` should go together and `write()` with `lseek()` may
   accompany them. So if number of related functions is grater than 2
   the interface creation is better.

But what if *all* calls should provide simulated errors? For free
functions on Linux platform function interception is possible. To do
this the following steps should be performed:

1. The main program should be dynamically linked. Any statically
   linked program doesn't involve `ld` in startup process so nobody
   handle next steps.

2. The required mock function implementations should be placed to
   shared object (dynamic library). For example, we want to intercept
   `open()` and `close()`. Our mock implementations should be placed
   as external functions to a library, for example, `libmock.so`.

3. The library with mocks is loaded as the very first library via
   setting to environment `LD_PRELOAD=libmock.so` before starting the
   program. For example, command might look like
   `LD_PRELOAD=libmocks.so ./sample`.

Sounds simple like "I dropped the atom bomb and it went off" (many
thanks to Harry Harrison) but involves many complex behaviors from
`ld`.

At the first the dynamically linked program is very similar to shared
object. So when dynamically linked program is started its startup is
performed via `ld`. This allows to control its behavior via
environment. The `LD_PRELOAD` is one of such environment variables.

The second aspect: it is possible to load multiple shared objects
which export the same symbol (i.e. function). The regular process of
calling function will call the function from the very first
library. Since libraries from `LD_PRELOAD` are loaded first even
before main program their symbols win in regular symbol lookup.

The third aspect: the mock implementation may require access to true
implementation. For example, the `open()` and `close()` calls are
intercepted but `read()` and `write()` are not. Since the program
will call them the mocked implementation of `open()` must return
proper file descriptor if it doesn't report simulated error. The same
is applicable to `close()`: it definitely must close proper file
descriptors via system call. So calls to true implementation in mocks
are imminent.

The fourth aspect: a nice pitfall is true symbols names. The C++
function names becomes mangled when exported from object file.

> TODO: insert sample of names

To
intercept them their mangled names should be used. This shouldn't be a
problem when `libmocks.so` is written in C++ but this not always
possible.

The fifth aspect: another pitfall is created by "glibc". Some
functions are versioned. During compilation the calls to versioned
functions are inserted depending on version of "glibc" used and
compile flags/definitions. So actual standard library contains set of
version implementations allowing to use single library to replace old
ones without recompilation of programs. As the result the mock
function should take care of versioning.

> TODO: insert sample from "glibc"

So the process of interception becomes as little bit complex.

Lets implement the same mocked `write()` call as in previous
section. The mocked `write()` will report error when write operation
covers LBA 8192 in file `/dev/sdt0`. To properly handle this the
mocking library needs to know which file descriptors refer to this
device. So `open()` call should be intercepted too. The  `creat()`,
`openat()`, `openat2()`, vector I/O, file descriptor duplication and
file system links handling are omitted for brevity and clarity.

Since `open()` is intercepted the `close()` should be intercepted too.

With all these requirements the implementation will look like:

1. List of known file descriptors which refer to our device.

2. Intercepted `open()` function which inspects a name and records
   file descriptor received from true implementation if file name is
   matches.

3. Intercepted `close()` function which removes file descriptor from
   list if it stored there. After that the true implementation is
   called.

4. Intercepted `write()` function. It will analyze file
   descriptor. If descriptor is stored in list the current position
   and I/O size is analyzed whether it covers "faulty" LBA. If faulty
   LBA is involved in I/O the `EIO` error is returned instead of
   calling true implementation.

> TODO: add sample

# Fault Injection
