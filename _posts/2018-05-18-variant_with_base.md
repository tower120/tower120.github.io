---
layout: post
title: variant with base class access
tag: [oop]
excerpt_separator: <!--more-->
---

<h2>Motivation</h2>

Have direct access to common base class of `std::variant`.

```c++
    struct Base{
        int pos;
    };
    struct A : Base{
        int a = 10;
    };
    struct B : Base{
        int b = 20;
    };
    
    std::variant<A,B> var;
    Base& base = std::get<Base>(var);    // can't do this
    // can do this, but this will cost you.
    Base& base = std::visit([](Base& base) -> Base& { 
        return base;
    }, var);
```

<h2>Final solution</h2>

```c++
    variant_w_base<Base, std::variant<A,B>> var;
    Base& base = *var.base();
    Base& base = std::get<Base>(var);
```
[source code](https://github.com/tower120/variant_w_base)
<!--more-->
<h2>Step-by-step</h2>

To allow base class access, we store pointer to base.
```c++
    template<class Base, class Variant>
    class variant_w_base{
        Base* m_base;
        Variant m_variant;

        void update_base(){
            m_base = std::visit([](auto&& arg) -> Base* {
                using Arg = std::decay_t<decltype(arg)>;
                if constexpr (std::is_same_v<Arg, std::monostate>){
                    return nullptr;
                }

                return static_cast<Base*>(&arg);
            }, m_variant);
        }
    }
```

Each time value changed, copied, moved we update base class pointer.

In order to work with `std::visit` , `std::get`, `std::get_if`, etc. we do specialize function templates. Like:

```c++
namespace std{

    template<class Visitor, class Base, class Variant>
    decltype(auto) visit(Visitor&& vis, NoteEditor::utils::variant_w_base<Base, Variant>& var){
        return std::visit(std::forward<Visitor>(vis), var.variant());
    };
    
}
```

<h2>Performance measurements</h2>

Accessing 100'000 element's base class data. (aka dumb linear search). 100 repeats:
```
std::variant   21ms
variant_w_base 2ms
```

[Benchmark](http://coliru.stacked-crooked.com/a/a93c75c3217d5657)

---

P.S. It is also possible to use it as virtual class  local storage, if you know all posible classes beforehand:

```c++
struct Interface{
    virtual int get() = 0;
}
struct A : Interface{
    virtual int do_smthg() override{ return 1; }
}
struct B : Interface{
    virtual int do_smthg() override{ return 2; }
}

variant_w_base<Interface, std::optional<std::monostate, A, B>> var;

var = A();
Interface* inteface = var.base();
inteface->get();
```