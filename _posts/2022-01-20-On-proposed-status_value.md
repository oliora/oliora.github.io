---
title: On proposed `status_value`
layout: post
tags: [c++, proposal, ranting]
---

There is proposal for [`status_value<Status, Value>`](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4233.html) class. This class is practically a syntax sugar on top of
`pair<Status, optional<Value>>`, in other words it is an optional value enriched with non-optional status.

The proposed design supposed to be used in the interface of proposed [concurrent queues](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0260r0.html), in particular for cases like this:

```cpp
enum class queue_op_status {success = 0, empty, full, closed, busy};

// Returns either queue_op_status::success and value or an error status and no value
template<class T>
auto queue<T>::pop() -> status_value<queue_op_status, T>;
```

The `status_value` proposal admits that proposed design is very close to long time proposed [`expected`](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p0323r10.html) but justifies the need for separate class like `status_value` by two points:
1. `expected` is designed to hold either a value or an unexpected error while the queue returns expected status so (as I understood it) using `expected` there would be conceptually wrong
2. `status_value` allows to have more than one status to be associated with a value, for example a status to indicate success with low contention and status to indicate success with high contention.

I think that justification for `status_value` does not hold well.

The first justification point looks like the author's misunderstanding of `expected` concept what is partially caused by `expected` wording that opposes expected value and unexpected error. In fact, `expected` is well suited for expected errors. It worth to mention that `status_value` also prioritizes value over status because its `operator bool` and `operator *` works over value. It's clear that both designs assume that the user is more interested in the value rather than in the error/status.

The second justification point is purely theoretical at the moment because the concurrent queue proposal does not require to return any status except for `success` along the value. Regardles of that, I think that success statuses that are returned along the value as extra information and error statuses that signal why value was not obtained should not be mixed in a single enum because they are completely different beasts. Mixing them together would contradict the fundamental concept that operation may return either a result or an error with both alternatives clearly separated from each other, the concept that C++ and most other programming languages have deeply adopted. Instead, in case if operation needs to return multiple success "statuses" I would prefer a combination of `expected` and `pair<Value, SuccessStatus>` so the pair become the expected result of the operation (similar to existing `map::insert` operation that returns `pair<iterator, bool>`):

```cpp
enum class queue_op_error {empty, full, closed, busy};
enum class contention_type {low, high};

template<class T>
auto queue::pop() -> expected<pair<T, contention_type>, queue_op_error>;
```

In conclusion, I don't think that there is a room for `status_value` as one of the a *standard fundamental* types for error handling. `status_value` does not replace `expected` so the latter will be eventually added anyway. In the meantime, `expected` class covers more use-cases including ones where `status_value` would be used so it's clearly more universal. And I'd rather see less number of error handling approaches in the C++ standard than other way around. Remember, what's added to the standard stays in the standard[^1]!


Update: some rewording to increase clarity.


[^1]: Rare exceptions exist
