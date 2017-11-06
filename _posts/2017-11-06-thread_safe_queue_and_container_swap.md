---
layout: post
title: thread-safe queue and container swap
tag: [thread-safety]
excerpt_separator: <!--more-->
---

Consider that you need queue with thread-safe `push_back` and `pop_front` operations. Also, consider that you want to consume all available elements, not just one. For example you need this for your home-grown LoopThreadExecutor, or some sort of pending queue:

```c++
template<class Message>
struct some_queue{
    void process(closure);
    void push(Message&& message);
}
```

<!--more-->

You may do this relativley efficently with just `std::vector` / `std::deque`, and `std::mutex` like this:

```c++
template<class Message>
struct some_queue{
    std::vector<Message> list;      // std::deque will work too
    std::mutex lock;

    // aka pop_back all :)
    template<class Closure>
    void process(Closure&& closure){
        std::vector<Message> my_list;
        {
            std::unique_lock l(lock);
            my_list = std::move(list);
            list.clear();
        }

        // process my_list
        for(auto& msg : my_list) closure(msg);
    }

    void push(const Message& message){
        std::unique_lock l(lock);
        list.emplace_back(message);
    }
}
```

Instead of read `list` under lock, we just swap it with empty one.
And now we have non-blocking read and ordered queue. We always have only one short-timed lock on read.

This is obviously MUCH faster than locking each element one by one or even locking whole container.

---

Theoretically, this can be improved for case where we can guarantee that `process` will always be called on the same thread, by using second container (aka double-buffer):

```c++
template<class container>
class double_buffer{
    container c1;
    container c2;
    container* c_active = &c1;
public:
    void swap(){ 
        if (c_active == &c1) c_active = &c2 
        else c_active = &c1; 
    }
    container& active() { return *c_active; }
}

...

    double_buffer<std::vector<Message>> buf;
    
    template<class Closure>
    void process(Closure&& closure){
        std::vector<Message>* my_list;
        {
            std::unique_lock l(lock);
            my_list = &buf.active();

            buf.swap();
            buf.active().clear();
        }

        // process my_list
        for(auto& msg : my_list) closure(msg);
    }
```

With this we reuse allocated memory in containers.

#### P.S.

More efficient than this may be [flat combining](http://mcg.cs.tau.ac.il/papers/spaa2010-fc.pdf) queue, for example one from the [cds library](https://github.com/khizmax/libcds). Though, I didn't compare them ;).