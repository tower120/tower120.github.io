---
layout: post
title: Functional std::lock, for complex cases
category: thread-safety
tag: [thread-safety, project-threading]
---

```c++
struct A{
  std::mutex lock;
  unsigned int value;
  B* b {nullptr};
}
struct B{
  std::mutex lock;
  unsigned int value;
  A* a {nullptr};
}

struct DataA{
  A a;  
  void swap_A_B_values();
} data_a;
struct DataB{
  B b;  
  void swap_B_A_values();
} data_b;
```

Consider that in `A` and `B`, all values accessed under lock.

Now, we want to swap `A::value`, with `B::value`. To do this, we need to lock both `A` and `B`. 

In `swap_A_B_values()` , we first need to lock A, then B.  
In `swap_B_A_values()` , we first need to lock B, then A. 

If we simply lock them from different threads, we'll get dead-lock.


We need use `std::lock` here, but we can't, because `std::lock` require both Lockables, beforehand. And to get second Lockable, we first must lock first. So...

You need [`lock_functional`][1]!

```c++
void DataA::swap_A_B_values() {
    auto l = lock_functional(
        [&](){ return &a.lock; },
        [&](){ return (a.b ? nullptr : &a.b->lock); }
    );
  if (!a.b) return;
  std::swap(a.value, a.b->value);
}
```
[`lock_functional`][1] accept closures as parameters. Closures must return pointer to Lockable, or null; if null - we stop trying (successfully locked Lockables will remain in locked state).

[`lock_functional`][1] do series of `try_lock`s in user defined order. And return tuple of `std::unqiue_lock`s. If one of the closures return null, `std::unqiue_lock` associated with this Lockable, and all next, will not own locks.

[`lock_functional`][1] a little bit unsafe, because it may occur in half-locked state, when one of the closures return `nullptr`. But sometimes, that half-locked state may be enough.

[`lock_all_functional`][1] return `std::optional< std::tuple< std::unique_lock > >` . It always have all locks in locked state, when `std::optional` have value.

```c++
void DataA::swap_B_A_values() {
    auto l = lock_all_functional(
        [&](){ return &b.lock; },
        [&](){ return (b.a ? nullptr : &b.a->lock); }
    );
  if (!l) return;
  std::swap(b.value, b.a->value);
}
```


[1]: https://github.com/tower120/threading/blob/master/src/threading/lock_functional.h