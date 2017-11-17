---
layout: post
title: Virtual inheritance without dynamic_cast
tag: [oop]
excerpt_separator: <!--more-->
---

**Motivation**. Extend class, that implement interface `I`, with interface derived from `I`, without virtual inheritance.

Have this:

```c++
// PURE interfaces per se. No data. All public.
struct IView{
    virtual void setOnClick() = 0;
};
struct ITextView : virtual IView {
    virtual void setText() = 0;
};

// implementation
struct View : virtual IView {
    virtual void setOnClick() override {
        std::cout << "setting OnClick!" << std::endl;
    }
};
struct TextView : virtual ITextView, virtual View {
    virtual void setText() override {
        std::cout << "setting text!" << std::endl;
    };
};
```

Without performance overhead of this:

```c++
IView* view;
ITextView* text_view = dynamic_cast<ITextView*>(view);
```

<!--more-->

## Solving the problem
#### Forwarding

First what come to mind is just forward all `IView` functions implemented by `View` to `TxtView`, like this:

```c++
struct TextView : ITextView, View {
    virtual void setText() override {
        std::cout << "setting text!" << std::endl;
    };
    virtual void setOnClick() override {
        View::setOnClick();
    }    
};
```

And that's ok. If we forget something to forward / implement, compiler will remind it. But if we have, like +8 functions in each level, most part of our classes will consists from forward functions. The higher inheritance level, the more functions we should forward.

#### Forwarding with macro

We need some machinery to forward base interface-implementing functions. Meta-programming with reflections would do a great help here, but... we don't have them... yet... So, for now, we can do forward macro for each interface, like:

```c++
struct IView{
    virtual void setOnClick() = 0;
};
#define forward_IView(To_class)\
    virtual void setOnClick() override {\
        To_class::setOnClick();\
    } 
```

And then :

```c++
struct TextView : ITextView, View {
    forward_IView(View)
    
    virtual void setText() override {
        std::cout << "setting text!" << std::endl;
    };
};
```

Let alone bogus process of writing macro like this, we have another problem. If we, by any reason, would want to override forwarded function, we obviously couldn't:

```c++
struct TextView : ITextView, View {
    forward_IView(View)
    
    virtual void setText() override {
        std::cout << "setting text!" << std::endl;
    };
    
    // compiler error, redifinition
    virtual void setOnClick() override {
       std::cout << "clicking" << std::endl;
    }     
};
```

## Solution

Have `Forward` helper per each interface:

```c++
struct IView : IBase {
    virtual void setOnClick() = 0;

    template<class T, class ...Interfaces>
    struct Forward : T, Interfaces... {
        virtual void setOnClick() override {
            T::setOnClick();
        }
    };
};
struct ITextView : IView {
    virtual void setText() = 0;

    template<class T, class ...Interfaces>
    struct Forward : IView::Forward<T, Interfaces...>  {
        virtual void setText() override {
            T::setText();
        }
    };
};
```

Use as:

```c++
struct TextView : IView::Forward<View, ITextView> {
    virtual void setText() override {
        std::cout << "setting text! i: " << std::endl;
    };
};
```

[Live example](http://coliru.stacked-crooked.com/a/773caa7443ece2bf)

You still need to write simple forward code, for each function in your interface. But now only once per interface. *[This can be completly automated, if you work with clang compiler fork  with experimental reflection/meta-programming support]*

Size-wise, solution equivalent to class with virtual-inheritance. Your interface classes MUST be zero-size.
