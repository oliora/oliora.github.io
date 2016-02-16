---
title: What's the difference between std::move(object.member) and std::move(object).member?
layout: post
tags: [c++, c++11, move, rvalue]
---

Imagine that you got a named temporary object and you would like to allow the move semantics is applied to the object's data member. You definitely need to force move semantics with `std::move()` but you have two options how to put it:

```cpp
void func(Object && object) {
    // Either
    do_something( std::move(object.member) ); // [1]
    // or
    do_something( std::move(object).member ); // [2]

    ...
}
```

Another example is a constructor:

```cpp
Foo::Foo(Object && object)
// Either
: bar( std::move(object.member) ) // [1]
// or
: bar( std::move(object).member ) // [2]
{}
```

The questions are: what is the difference between expression `std::move(object.member)` and `std::move(object).member`
and which one should you use?

## What is the difference?

First expression `std::move(object.member)` casts the result of a class member access to an *rvalue*, so the result of the
expression is **always** a [*cv*-qualified] *rvalue*. (Note that *cv* means "const volatile")

Second expression `std::move(object).member` casts the object to a [*cv*-qualified] *rvalue* first then
performs a class member access, so it literally means "a member of an rvalue". And here the C++11 member access rules
take the wheel. The rules are described in [[expr.ref/4]](http://eel.is/c++draft/expr.ref#4) of the standard.
From there we can figure out the following:

- if `member` is a reference then expression `std::move(object).member` is a [*cv*-qualified] *lvalue*,
- otherwise if `member` is a static member then the expression is a [*cv*-qualified] *lvalue*,
- otherwise the expression is a [*cv*-qualified] *rvalue*.
  (An [*xvalue*](http://en.cppreference.com/w/cpp/language/value_category) if you want to be more precise)
  
It's quite easy to remember: non-static non-reference data members are part of the object (it's a *composition*), so they are the same value category as the object itself, i.e. *rvalues* if the object is an *rvalue*.
In contrast, things pointed by a reference members and static data members don't belong to the object thus ones don't depend on the object's value category, ones are always *lvalues*.

## Which one to use?

The first expression `std::move(object.member)` is much stronger than the second one.
Use it when you need to **force** move semantics for the member itself whatever the member is and you really understand all the consequences of it. (Here imagine an unexpected move from a shared object!).

The second expression `std::move(object).member` is safer and more intelligent: it forces only the object itself to became an *rvalue*, then (the safe) member access rules do the rest. Rather than force move semantics for the member, the expression gently **allows** it.
It is safe operation as long as you know that the object is a temporary, doesn't matter what the member is.
C++11 member access rules designed to be safe so you can rely on them.

Let's talk a bit more about the expressions safety.
There are many cases when both expressions give you the same result so you might think it doesn't matter which one to use in a such case. Be careful with this assumption: what is safe now can crash your application after a small innocent refactoring!

Assume that we have the following:

```cpp
struct WidgetCfg {
    std::string config_name_;  // non-reference non-static data member
    ...
};

struct Window {
    std::string config_name_;
    
    Window(WidgetCfg && w) // Constructor accepting a temporary WidgetCfg so we are safe to move from it
    // Either
    : config_name_(std::move(w.config_name_)) // [1] move from temporary. No problem
    // or
    : config_name_(std::move(w).config_name_) // [2] move from temporary. No problem
    {}
    
    ...
};
```

As you can see, both expressions do the same and look safe at the moment.

Now we decided to optimize a memory usage for application-wide `config_name_` so we want to spread references to a shared global string rather than one's copies.
We've changed `WidgetCfg` class definition but forget about `Window` class, so we have the following:

```cpp
struct WidgetCfg {
    std::string & config_name_;  // Now it is a reference to a shared global string
    ...
};

struct Window {
    std::string config_name_;  // Oops, we forget to change this
    
    Window(WidgetCfg && w) // Constructor accepting a temporary WidgetCfg so we are safe to move from it
    // Either
    : config_name_(std::move(w.config_name_)) // [1] move from the global variable, leaves one in the
                                              //     (empty or even dangerous to use) 'moved from' state!
    // or
    : config_name_(std::move(w).config_name_) // [2] not a temporary any more, so makes a copy rather than move.
                                              //     Not so effective but still safe!
    {}
    
    ...
};
```

Thus whenever it's possible, instead of `std::move(object.member)` use `std::move(object).member`, because the latter one stays safe
no matter how the member definition is changed.


UPD

There are more discussions on the post topic:

- [A tentative notion of move-independence](http://ldionne.com/2016/02/17/a-tentative-notion-of-move-independence/) post by Louis Dionne
- [Reddit](https://www.reddit.com/r/cpp/comments/45w3fs/whats_the_difference_between_stdmoveobjectmember/)
