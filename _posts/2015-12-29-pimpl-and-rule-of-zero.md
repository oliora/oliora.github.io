---
title: PIMPL, Rule of Zero and Scott Meyers
layout: post
tags: [meyers, c++, c++11, pimpl, idioms]
---

Some of my readers may say that mentioning Scott Meyers in the title is a dirty trick, and I partially agree :)
The idea of this post came to me when I was reading Scott Meyers' great book
[Effective Modern C++](http://www.amazon.com/Effective-Modern-Specific-Ways-Improve/dp/1491903996).
In this post I try to improve Scott Meyers's C++11 PIMPL (The book's item 22: *When using the Pimpl Idiom, define special member functions in the implementation file*)
using an idea taken from the Rule of Zero.

Table of contents:

* [Classic PIMPL](#classic-pimpl)
* [Scott Meyers' C++11 PIMPL](#scott-meyers-c11-pimpl)
* [Rule Of Zero](#rule-of-zero)
* [PIMPL Without Special Members Defined](#pimpl-without-special-members-defined)
 * [Noncopyable PIMPL](#noncopyable-pimpl)
 * [Movable And Copyable PIMPL](#movable-and-copyable-pimpl)

**If you're familiar with the Rule of Zero and Scott Meyer's C++11 PIMPL then I advice you to go directly to the final part [PIMPL Without Special Members Defined](#pimpl-without-special-members-defined).**

If you're just looking for the SPIMPL library (described below), here is the link:
[spimpl.h](https://github.com/oliora/samples/blob/master/spimpl.h).

I want to thank **Maxim Pudov** for his help in preparing this post!
Without his reviews and comments it would be much more messy.


## Classic PIMPL

You certainly know the [Pointer to IMPLementation](https://en.wikipedia.org/wiki/Opaque_pointer) (PIMPL) idiom.
It is about hiding the actual class implementation behind the pointer to forward declared *implementation class*,
so all the implementation's heavy details and dependencies don't pollute the class interface and its header file.
There is a classic C++98 PIMPL example:

Header file:

```cpp
class Parser {
public:
    Parser(const char *params);
    ~Parser();

    void parse(const char *input);
    
    // Other methods related to responsibility of the class
    ...

private:
    Parser(const Parser&);              // noncopyable
    Parser& operator=(const Parser&);   //

    class Impl;     // Forward declaration of the implementation class
    Impl *impl_;    // PIMPL
};
```

Source file:

```cpp
// Include all headers the implementation requires

// The actual implementation definition:
class Parser::Impl {
public:
	Impl(const char *params) {
		// Actual initialization
		...
	}
	
    void parse(const char *input) {
        // Actual work
        ...
    }
};

// Create an implementation object in ctor
Parser::Parser(const char *params)
: impl_(new Impl(params))
{}

// Delete the implementation in dtor
Parser::~Parser() { delete impl_; }

// Forward an operation to the implementation
void Parser::parse(const char *input) {
    impl_->parse(input);
}

// Forward other operations to the implementation
...
```

In C++11 you may also declare (in the header) and implement (in the source file) a move constructor and a move assignment operator.

## Scott Meyers' C++11 PIMPL

In his exclusive book, Scott Meyers proposed to use `std::unique_ptr` and `std::shared_ptr` (for unique and shared implementations respectively) instead of raw pointers to implement PIMPLs in a modern way. I will not touch the shared implementation because it is perfectly explained in the book and what you need is just to follow the advice. The unique implementation case is more controversial. Here is how it is proposed in the book (please note that the code sample was not exactly taken from the book, it just brings the idea from):

Header:

```cpp
#include <memory>

class Parser {
public:
    Parser(const char *params);
    ~Parser();
    
    Parser(Parser && op) noexcept;              // movable and noncopyable
    Parser& operator=(Parser && op) noexcept;   //

    void parse(const char *input);
    ...

private:
    class Impl;                     // Forward declaration of the implementation class
    std::unique_ptr<Impl> impl_;    // PIMPL
};
```

`Parser` class is noncopyable because defining either move ctor or move assignment operator disables generation of copy ctor/assgnment.

Source file:

```cpp
// Include all headers the implementation requires

// The actual implementation definition:
class Parser::Impl { ... };

// Create an implementation object
Parser::Parser(const char *params)
: impl_(new Impl(params))
{}

// Tell the compiler to generate default special members which utilize
// the power of std::unique_ptr.
// We can do it here because the implementation class is defined at this point thus
// std::unique_ptr can properly handle the implementation pointer.
Parser::~Parser() = default;
Parser::Parser(Parser &&) noexcept = default;
Parser& Parser::operator=(Parser &&) noexcept = default;

// Forward an operation to the implementation
void Parser::parse(const char *input) {
    impl_->parse(input);
}
...

```

As you can see, the idea behind it is to use `std::unique_ptr` to handle ownership of the implementation. The only problem we have to solve is that special members of `std::unique_ptr` unable to handle (to delete, to be more precise) incomplete types thus they must be instantiated at the point where the implementation class is defined. So we force them to be instantiated in the source file rather than in the header. To do so we define `Parser` class special members in the source file.

But what if you need a movable and copyable class without sharing the implementation? Here how it is proposed in the book:

Header:

```cpp
#include <memory>

class CopyableParser {
public:
    CopyableParser(const char *params);
    ~CopyableParser();
    
    CopyableParser(CopyableParser && op) noexcept;              // movable
    CopyableParser& operator=(CopyableParser && op) noexcept;   //

    CopyableParser(const CopyableParser& op);                   // and copyable
    CopyableParser& operator=(const CopyableParser& op);        //
    
    void parse(const char *input);
    ...

private:
    class Impl;                     // Forward declaration of the implementation class
    std::unique_ptr<Impl> impl_;    // PIMPL
};
```

Source file:

```cpp

// All the same as in previous example
...

// Implementation of a copy constructor/assignment

CopyableParser::CopyableParser(const CopyableParser& op)
: impl_(new Impl(*op.impl_))
{}

CopyableParser& CopyableParser::operator=(const CopyableParser& op) {
    if (this != &op) {
        impl_.reset(new Impl(*op.impl_));
    }
    return *this;
}
```

The same approach plus a copy constructor and a copy assignment are added. Both members perform cloning of the implementation.

I like Scott Meyers' approach but I would prefer to eliminate all this boring boilerplate code of special members.

## Rule of Zero

You probably heard about [Rule of Zero](https://rmf.io/cxx11/rule-of-zero/) introduced by R. Martinho Fernandes.
The rule is an enhancement (I would say, a replacement) of old-good
[rule of three](https://en.wikipedia.org/wiki/Rule_of_three_%28C%2B%2B_programming%29),
which is more often called the rule of five nowadays. The root idea of the Rule of Zero is quite simple:

__A class should have a definition of *either all* special member functions *or zero* of them.__

Note that there are five special member functions we're talking about: destructor, copy/move constructor and copy/move assignment.
So if a class needs to deal with tricky resource management/ownership
(that's all the mentioned special member functions about) then this ownership should be extracted
to a dedicated class where custom special member functions are defined.
Other classes should just include first class as a member and rely on _compiler-generated_ special member functions.
Clear, simple and great! Personally, I think that the Rule of Zero is one of the greatest idioms introduced in C++ world in the latest years.

## PIMPL Without Special Members Defined

It would be nice if our PIMPL can have no special member functions defined, so be on "zero side" of the Rule of Zero :)
Below is my proposal on how to achieve this.

### Noncopyable PIMPL

The idea is simple: let's utilize `std::unique_ptr`'s feature to store a pointer to custom deleter.
With such a trick `unique_ptr`'s special members can handle an incomplete type so we may allow a compiler to *implicitly* generate all the special members for us right in the header. That's just a little twist from Scott Meyers' approach but it gives a great advance.

Header:

```cpp
#include <memory>

class Parser {
public:
    Parser(const char *params);

    // Destructor, move constructor and move assignment are compiler-generated.
    // Copy constructor and copy assignment are implicitly deleted.
    
    void parse(const char *input);
	...

private:
    class Impl; // Forward declaration of the implementation class.
    
    // Stores the implementation and the implementation's deleter as well.
    // Deleter is a pointer to a function with signature `void func(Impl *)`.
    std::unique_ptr<Impl, void (*)(Impl *)> impl_;
};
```

Source file:

```cpp
// Include all headers the implementation requires

// The actual implementation definition:
class Parser::Impl { ... };

// Create an implementation object with custom deleter
Parser::Parser(const char *params)
: impl_(
    new Impl(params),
    [](Impl *impl) { delete impl; })
{}

// Forward an operation to the implementation
void Parser::parse(const char *input) {
    impl_->parse(input);
}
...
```

As you can see the amount of boilerplate code is significantly reduced but still requires
to repeat a non-trivial pointer type and a custom deleter for each PIMPL class. Let's eliminate this.

Here let me to introduce SPIMPL (Smart Pointer to IMPLementation) - a small header-only C++11 library with aim to simplify the implementation of PIMPL idiom. It is a single header library you can get here:
[spimpl.h](https://github.com/oliora/samples/blob/master/spimpl.h).

`spimpl::unique_impl_ptr`:

```cpp
namespace spimpl {
    namespace details {
        template<class T>
        void default_delete(T *p) noexcept {
            static_assert(sizeof(T) > 0, "default_delete cannot delete incomplete type");
            static_assert(!std::is_void<T>::value, "default_delete cannot delete incomplete type");
            delete p;
        }
    }
    
    // Pointer to unique implementation
    template<class T, class Deleter = void(*)(T*)>
    using unique_impl_ptr = std::unique_ptr<T, Deleter>;

    // Constructs an object of type T and wraps it and related default deleter in `unique_impl_ptr`
    template<class T, class... Args>
    inline unique_impl_ptr<T> make_unique_impl(Args&&... args) {
        static_assert(!std::is_array<T>::value, "unique_impl_ptr does not support arrays");

        return unique_impl_ptr<T>(
            new T(std::forward<Args>(args)...),
            &details::default_delete<T>);
    }
}
```

Header:

```cpp
#include <memory>
#include <spimpl.h>

class Parser {
public:
    Parser(const char *params);

    // Destructor, move constructor and move assignment are compiler-generated.
    // Copy constructor and copy assignment are implicitly deleted.

    void parse(const char *input);
    ...
    
private:
    class Impl;
    spimpl::unique_impl_ptr<Impl> impl_;  // Movable Smart PIMPL
};
```

Now declaration of the implementation pointer is simple.
A construction of an implementation is simple as well:

```cpp
Parser::Parser(const char *params)
: impl_(spimpl::make_unique_impl<Impl>(params))
{}
```

### Movable And Copyable PIMPL

Let's implement a special smart pointer class `spimpl::impl_ptr` for movable and copyable types.
It based on `spimpl::unique_impl_ptr`. In addition to a *deleter* it contains a *copier*.
The *copier* is a callable which makes a (heap allocated) clone of the implementation object, like:

```cpp
template<class T>
T *default_copy(T *src) {
    static_assert(sizeof(T) > 0, "default_copy cannot copy incomplete type");
    static_assert(!std::is_void<T>::value, "default_copy cannot copy incomplete type");
    return new T(*src);
}
```

**Significantly simplified** `spimpl::impl_ptr`
(check [spimpl.h](https://github.com/oliora/samples/blob/master/spimpl.h) for complete source code):

```cpp
namespace spimpl {
    namespace details {
        template<class T>
        void default_delete(T *p) noexcept { ... }

        template<class T>
        T *default_copy(T *src) { ... }
    }

    template<class T, class Deleter = void (*)(T*), class Copier = T*(*)(T*)>
    class impl_ptr {
    public:
        template<class U, class D, class C>
        impl_ptr(U *u, D&& deleter, C&& copier) noexcept
        : ptr_(u, std::forward<D>(deleter))
        , copier_(std::forward<C>(copier))
        {}
        
        impl_ptr(const impl_ptr& r) : impl_ptr(r.clone()) {}

        impl_ptr& operator= (const impl_ptr& r) {
            if (this == &r)
                return *this;
            return operator=(r.clone());
        }

        impl_ptr(impl_ptr&&) noexcept = default;

        impl_ptr& operator= (impl_ptr&&) noexcept = default;
        
        impl_ptr clone() const {
            return impl_ptr(
                ptr_ ? copier_(ptr_.get()) : nullptr,
                ptr_.get_deleter(),
                copier_);
        }
        
        ...

    private:
        std::unique_ptr<T, Deleter> ptr_;
        Copier copier_;
    };
    
    // Constructs an object of type T and wraps it and related
    // default deleter and copier in `impl_ptr`
    template<class T, class... Args>
    inline impl_ptr<T> make_impl(Args&&... args) {
        return impl_ptr<T>(
            new T(std::forward<Args>(args)...),
                &details::default_delete<T>,
                &details::default_copy<T>);
    }
}
```

Example of using of `spimpl::impl_ptr` to implement movable and copyable PIMPL:

Header:

```cpp
#include <memory>
#include <spimpl.h>

class CopyableParser {
public:
    CopyableParser(const char *params);

    // All five special members are compiler-generated.

    void parse(const char *input);
    ...
    
private:
    class Impl;
    spimpl::impl_ptr<Impl> impl_;  // Movable and copyable Smart PIMPL
};
```

Creation of an implementation in the source file is easy, and no boilerpate code is needed:

```cpp
CopyableParser::CopyableParser(const char *params)
: impl_(spimpl::make_impl<Impl>(params))
{}
```

Looks better, isn't it?
