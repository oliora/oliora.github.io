---
title: PIMPL, Rule of Zero and Scott Meyers
layout: post
---

# PIMPL, Rule of Zero and Scott Meyers

Some of my readers may say that mentioning Scott Meyers in the title is a dirty trick, and I partially agree.
For my defence I want to say that the idea of this post came to me when I was reading Scott Meyers' great book, [Effective Modern C++](http://www.amazon.com/Effective-Modern-Specific-Ways-Improve/dp/1491903996).
And even more, this post is a further development of the book's Item 22: *When using the Pimpl Idiom, define special member functions in the implementation file*. The post is about applying the Rule of Zero to Scott Meyers advise on C++11 PIMPLs.
You will find more on this below. Let's start from the basics first!

Table of contents:

- [PIMPL](#pimpl)
- [Scott Meyers' C++11 PIMPL](#scott-meyers-c11-pimpl)
- [Rule Of Zero](#rule-of-zero)
- [PIMPL Without Special Members Defined](#pimpl-without-special-members-defined)

**If you already familiar with the rule of zero and Scott Meyer's C++11 PIMPL then go directly to a final part [PIMPL Without Special Members Defined](#pimpl-without-special-members-defined).**

You can download a complete SPIMPL (more on this below) header here:
[spimpl.h](https://github.com/oliora/samples/blob/master/spimpl.h).

## PIMPL

Most of you know the [Pointer to IMPLementation](https://en.wikipedia.org/wiki/Opaque_pointer) (PIMPL) idiom.
It is about hiding of actual class implementation under the pointer to forward declared implementation class so all the implementation's heavy details and dependancies don't pollute the class interface thus it is simple and its header is clear and lightweight.
There is a classical C++98 PIMPL example:

Header file:

```cpp
class Protocol {
public:
    Protocol();
    ~Protocol();

    void foo();

private:
    Protocol(const Protocol&);              // noncopyable
    Protocol& operator=(const Protocol&);   //

    class Impl;                             // Forward declaration of the implementation class
    Impl *impl_;                            // PIMPL
};
```

Source file:

```cpp
// Include all headers the implementation requires

// The actual implementation definition:
class Protocol::Impl {
public:
    void foo() {
        // Do actual work
        ...
    }
};

// Create an implementation object in ctor
Protocol::Protocol(): impl_(new Impl()) {}

// Delete the implementation in dtor
Protocol::~Protocol() { delete impl_; }

// Forward operation to the implementation
void Protocol::foo() {
    impl_->foo();
}
```

In C++11 you may also declare (in the header) and implement (in the source file) a move constructor and a move assignment operator.

## Scott Meyers' C++11 PIMPL

In his exclusive book, Scott Meyers proposed to use `std::unique_ptr` and `std::shared_ptr` (for unique and shared implementations respectively) instead of raw pointers to implement PIMPL in a modern way. I will not touch the shared implementation because it is perfectly explained in the book and what you need is just to follow the book's advise. The unique implementation case is more controversial. Here is how it is proposed in the book (please note that the code sample is not exactly taken from the book, it just brings the idea from):

Header:

```cpp
#include <memory>

class Protocol {
public:
    Protocol();
    ~Protocol();
    
    Protocol(Protocol && op) noexcept;              // movable and noncopyable
    Protocol& operator=(Protocol && op) noexcept;   // (copy ctor/assignment are implicitly deleted)

    void foo();

private:
    class Impl;                             // Forward declaration of the implementation class
    std::unique_ptr<Impl> impl_;            // PIMPL
};
```

Source file:

```cpp
// Include all headers the implementation requires

// The actual implementation definition:
class Protocol::Impl { ... };

// Create an implementation object
Protocol::Protocol(): impl_(new Impl()) {}

// Forward operation to the implementation
void Protocol::foo() {
    impl_->foo();
}

// Ask compiler to use default special members which utilize the power of std::unique_ptr.
// We can do it here because the implementation class is defined at this point thus
// std::unique_ptr can properly handle the implementation pointer.
Protocol::~Protocol() = default;
Protocol::Protocol(Protocol &&) noexcept = default;
Protocol& Protocol::operator=(Protocol &&) noexcept = default;
```

As you can see, the idea is to use `std::unique_ptr` to handle ownership of the implementation. The only problem we need to solve is that special members of `std::unique_ptr` unable to handle (to delete, to be more precise) incomplete types thus they must be instantiated at the point where the implementation class is defined. So we force them to be instantiated in the source file rather than in the header. To do so we define `Protocol` class special members in the source file.

But what if you need a movable and copyable class without sharing the implementation? Here how it is proposed in the book:

Header:

```cpp
#include <memory>

class CopyableProtocol {
public:
    CopyableProtocol();
    ~CopyableProtocol();
    
    CopyableProtocol(CopyableProtocol && op) noexcept;              // movable
    CopyableProtocol& operator=(CopyableProtocol && op) noexcept;   //

    CopyableProtocol(const CopyableProtocol& op);                   // and copyable
    CopyableProtocol& operator=(const CopyableProtocol& op);        //
    
    void foo();

private:
    class Impl;                             // Forward declaration of the implementation class
    std::unique_ptr<Impl> impl_;            // PIMPL
};
```

Source file:

```cpp

// All the same as in previous example
...

// Implementation of a copy constructor/assignment

CopyableProtocol::CopyableProtocol(const CopyableProtocol& op)
: impl_(new Impl(*op.impl_))
{}

CopyableProtocol& CopyableProtocol::operator=(const CopyableProtocol& op) {
    if (this != &op) {
        impl_.reset(new Impl(*op.impl_));
    }
    return *this;
}
```

The same approach plus a copy constructor and a copy assignment are added. Both members perform cloning of the implementation.

I like Scott Meyers' approach but I would prefer to eliminate all this boring special members boilerplate.

## Rule of Zero

Many of you probably heard about [Rule of Zero](https://rmf.io/cxx11/rule-of-zero/) introduced by R. Martinho Fernandes.
The rule is an enhancement (I would say, a replacement) of old-good [rule of three](https://en.wikipedia.org/wiki/Rule_of_three_%28C%2B%2B_programming%29),
which is more correctly to call the rule of five in C++11. The root idea of the rule of zero is quite simple:

__A class should have a definition of *either all* special member functions *or zero* of them.__

Note that there are five special member functions we're talking about: destructor, copy/move constructor and copy/move assignment.
So if a class needs to deal with tricky resource management/ownership
(that's all the mentioned special member functions about) then this ownership should be extracted
to a dedicated class where all needed special member functions are _user-defined_.
Other classes should just include first class as a member and rely on a _compiler-generated_ special member functions.
Clear, simple and great! Personally, I think that the rule of zero is one of the greatest idioms introduced in C++ world in the latest years.

## PIMPL Without Special Members Defined

It would be nice if our PIMPL can have no special member functions defined, so be on "zero side" of the rule of zero :)
Below I present my approach how to acheive this.

### Noncopyable PIMPL

The idea is simple: let's utilize `std::unique_ptr`'s feature to store a pointer to custom deleter. With such trick `unique_ptr`'s special members can handle an incomplete type so we may allow a compiler to *implicitly* generate all the special members for us:

Header:

```cpp
#include <memory>

class Protocol {
public:
    Protocol();

    void foo();
    
    // Note that all the special members are compiler-generated (or deleted if cannot be generated)
    // In this case, copy 

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
class Protocol::Impl { ... };

// Forward operation to the implementation
void Protocol::foo() {
    impl_->foo();
}

// Create an implementation object with custom deleter
Protocol::Protocol(): impl_( new Impl(), [](Impl *impl) { delete impl; } ) {}
```

As you can see the amount of boilerplate code is significantly reduced. This approach can be generalized to be simpler to use:

`spimpl.h` (Smart Pointer to IMPLementation :):
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
 
    // Make unique implementation in more pleasant and safe way
    template<class T, class... Args>
    inline unique_impl_ptr<T> make_unique_impl(Args&&... args) {
        static_assert(!std::is_array<T>::value, "unique_impl_ptr does not support arrays");
        return unique_impl_ptr<T>(new T(std::forward<Args>(args)...), &details::default_delete<T>);
    }
}
```

Header:

```cpp
#include <memory>
#include <spimpl.h>

class Protocol {
public:
    Protocol();

    void foo();
    
    // All the special members are compiler-generated!
    // In this case three of them are generated: destructor, move constructor and move assignment.
    // Copy constructor and copy assignment are implicitly deleted because `spimpl::unique_impl_ptr`
    // is not copyable.

private:
    class Impl;                             // Forward declaration of the implementation class.
    spimpl::unique_impl_ptr<Impl> impl_;    // Movable Smart PIMPL
};
```

Creation of an implementation in the source file is easy:

```cpp
Protocol::Protocol(): impl_( spimpl::make_unique_impl<Impl>() ) {}
```

### Movable And Copyable PIMPL

We may go further and implement a special smart pointer class `spimpl::impl_ptr` for movable and copyable types. It is based on `std::unique_ptr` and holds deleter and copier as well. The copier is a callable which returns a pointer to the argument's clone, like:

```cpp
        template<class T>
        T *default_copy(T *src) {
            static_assert(sizeof(T) > 0, "default_copy cannot copy incomplete type");
            static_assert(!std::is_void<T>::value, "default_copy cannot copy incomplete type");
            return new T(*src);
        }
```

**Simplified** `spimpl::impl_ptr`:

```cpp
namespace spimpl {
    namespace details {
        template<class T>
        void default_delete(T *p) noexcept { ... }

        template<class T>
        T *default_copy(T *src) { ... }
    }

    template<class T, class Deleter = void (*)(T*), class Copier = T*(*)(T*)>
    class impl_ptr
    {
    public:
        template<class U, class D, class C>
        impl_ptr(U *u, D&& deleter, C&& copier) noexcept
        : ptr_(u, std::forward<D>(deleter))
        , copier_(std::forward<C>(copier))
        {}
        
        impl_ptr(const impl_ptr& r) : impl_ptr(r.clone()) {}

        impl_ptr(impl_ptr&&) noexcept = default;

        impl_ptr& operator= (const impl_ptr& r) {
            if (this == &r)
                return *this;
            return operator=(r.clone());
        }

        impl_ptr& operator= (impl_ptr&&) noexcept = default;
        
        impl_ptr clone() const {
            if (!ptr_)
                return impl_ptr(nullptr, ptr_.get_deleter(), copier_);
            else
                return impl_ptr(copier_(ptr_.get()), ptr_.get_deleter(), copier_);
        }

    private:
        std::unique_ptr<T, Deleter> ptr_;
        Copier copier_;
    };
    
    template<class T, class... Args>
    inline impl_ptr<T> make_impl(Args&&... args)
    {
        return impl_ptr<T>(new T(std::forward<Args>(args)...), &details::default_delete<T>, &details::default_copy<T>);
    }
```

**If you are interested in a complete implementation, you can find it here:
[spimpl.h](https://github.com/oliora/samples/blob/master/spimpl.h).**

Example of using of `spimpl::impl_ptr` to implement movable and copyable PIMPL:

Header:

```cpp
#include <memory>
#include <spimpl.h>

class CopyableProtocol {
public:
    CopyableProtocol();

    void foo();
    
    // All the special members are compiler-generated!
    // In this case it will be all five members: destructor, move/copy constructors and move/copy assignment
    // because `spimpl::impl_ptr` is movable and copyable.

private:
    class Impl;                      // Forward declaration of the implementation class.
    spimpl::impl_ptr<Impl> impl_;    // Movable and copyable Smart PIMPL
};
```

Creation of an implementation in the source file is easy:

```cpp
CopyableProtocol::CopyableProtocol(): impl_( spimpl::make_impl<Impl>() ) {}
```

Looks nice, isn't it?
