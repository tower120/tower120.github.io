---
layout: post
title: Expandable / runtime enum class
tag: [oop]
excerpt_separator: <!--more-->
---

Sometimes you need enum, and don't know exact set beforehand. 

One could just use `int` in conjunction with counter. But it would not be typesafe. Of course, its is better to wrap enum's id and counter in a dedicated class. Here is my attempt of this concept generalization:

```c++
struct Shape : Enum<Shape, char>{};

namespace Shapes{
    static const AddEnum<Shape> circle;
    static const AddEnum<Shape> triangle;
}
```

latter somewhere...

```c++
#include "Shape.h"

namespace Shapes{
    static const AddEnum<Shape> square;
}
```

C++ doesn't have concept of class extending (you can't add class members without inheritance), like [C# extension methods](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods), or [Swift extensions](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Extensions.html). So we utilize namespaces for this purpose.

Usage:

```c++
Shape active_shape = Shapes::square;

void draw(Shape shape){
    if (active_shape == shape) return;
    active_shape = shape;
    
    do_draw();
}
```

The downside of this method - you can't use `switch` with it. (`switch` work with constexpr values only).

Alternatively it can look like:

```c++
namespace Shape{
    struct type : Enum<type, char>{};

    static const AddEnum<type> circle;
    static const AddEnum<type> triangle;
}

...

namespace Shape{
    static const AddEnum<type> square;
}

...

Shape active_shape = Shape::square;

void draw(Shape::type shape){
    if (active_shape == shape) return;
    active_shape = shape;
    
    do_draw();
}
```


[Live example](http://coliru.stacked-crooked.com/a/27aa4e54b80d3104)

---

Implementation code:

```c++
template<class type>
struct Counter {
    static_assert(std::is_integral_v<type>);
    std::atomic<type> last = std::numeric_limits<type>::min();

    [[nodiscard]]
    type operator()(){
        const type was = last.fetch_add(1, std::memory_order_acq_rel);
        assert(was < std::numeric_limits<type>::max() );
        return was;
    }
};

template<class Derived, class type>
class Enum{
protected:
    inline static Counter<type> counter;
    type m_id;
public:
    operator type() const {
        return m_id;
    }

    bool operator==(const Derived& other) const{
        return m_id == other.m_id;
    }
    bool operator!=(const Derived& other) const{
        return m_id != other.m_id;
    }
};

template<class EnumT>
class [[nodiscard]] AddEnum final : public EnumT{
public:
    AddEnum(){
        this->m_id = this->counter();
    }    
};
```