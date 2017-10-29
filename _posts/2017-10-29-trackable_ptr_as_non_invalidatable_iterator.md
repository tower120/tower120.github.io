---
layout: post
title: trackable_ptr + std::vector = non-invalidatable iterator
tag: [project-trackable-ptr]
excerpt_separator: <!--more-->
---

Thou it is obvious, that using [`trackable_ptr`][1] allow  container's element access without need of iterator,or index. It may not come at first glance, that you can use [`trackable_ptr`][1] to do same things, you do with regular iterator (or index access) of any container with contiguous elements storage, like `std::vector`.
<!--more-->

`std::vector` being contiguous, allow us, to calculate index element from its pointer:
```c++
template<class T>
std::size_t get_index(T* ptr_to_element, std::vector<T>& vec){
    return ptr_to_element - vec.dat();
}
```
And as you can see, it is pretty fast. [https://godbolt.org/g/PiyE45](https://godbolt.org/g/PiyE45)

It is not very useful, when dealing with regular pointers, because they invalidates, after almost each interaction with `std::vector`. But [`trackable_ptr`][1] - is another thing. It always have pointer to element or null, if element was destroyed. So we can get iterator from [`trackable_ptr`][1]:

```c++
template<class T, class R>
auto get_iterator(const std::vector<T>& vec, const trackable_ptr<R>& ptr){
    return vec.begin() + get_index(ptr.get(), vec);
}
```
Now we can use [`trackable_ptr`][1] to do anything, we can do with `std::vector`s iterator:
```c++
    // erase
    vec.erase(get_iterator(vec, my_trackable_ptr));

    // insert after element
    vec.insert(get_iterator(vec, my_trackable_ptr), 3);
```

[Source code][1]


[1]:https://github.com/tower120/trackable_ptr
