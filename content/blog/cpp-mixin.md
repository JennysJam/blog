+++
title = "An idea for C++ Mixins"
date = 2024-11-16
description = "An idea for a mixin/trait like system for C++"
slug="cpp-mixin"
+++

# A C++ Mixin System

I've had this idea in the back of my head for a while of pervasively using mixins to add code and logic for more high level concepts, although this gets somewhat close to Rust style traits (and C++ concepts). This has existed in the back of my head for a long time as a way to model a framework or standard library implementaiton while also providing it for user types.

I think the big asterick to all of this design is that my ideal framework would not look like standard C++ but like a slightly weirder Rust stdlib:

* Errors are signified through a `Result<>` or `Option<>` like class, and both can handle reference like payload -- instead of exceptions, you can abort threads or the entire program.
* All constructors that aren't default, copy or move are 'data constructors' (each parameter is simply initializing the equivalent field)
* Non-trivial object creation is all done via a `create()` factory funtion (or `make()`, or w.e.) that returns an `Result` value.
* All non-trivial objects that can be moved or copied have a default constructor that initializes objects into a [NOP](https://en.wikipedia.org/wiki/Null_object_pattern) like state -- C++ sort of implicitly requires them for a lot of stuff, so I think owning this and making it more of a first class concept would be nice. 
* Any non-trivial interaction with an object in a NOP state will trigger an assertion error.
* Most methods that can fail but you'd like to go for happy path (like `clone()` or `serialize()` below) will abort if an error occurs, and they will have associated `cloneTry()` or `serializeTry()` methods that return an `Result` -- implicitly i'm imaging most inner plubming methods return these.
* Aside from function paramters, pointers are wrapped in "utility types" which are just newtypes that better tie in what their semantics are (e.g. pointer to unsized array) that could allow the 'default' expected behavior for moving them.

I'm writing these down because I don't ever expect to implement this but my dream can at least be expressed in prose \:\).

[Mixins](https://en.wikipedia.org/wiki/Mixin) are a way to add code to an object or class without going through the normal inheritance system. I think viewed through that lens what I'm doing specifically isn't, but it feels close enough to make it 'count'?

## My example: a `Clone` trait

Borrowing an idea from [Rust](https://doc.rust-lang.org/std/clone/trait.Clone.html) where objects which represent handles to resources (including memory resources) can't be deep copied without explicit usage.

So the requiresments here are:

* A `clone()` method that creates a new vesion of an object.

Bonuses:

* Ability to assert at compile time that a class implements this interface 
* A standalone `clone()` function, more as a demonstration than anything.

I think the standard C++ interface method would look like this:

```cpp
class IClone
{
public:
    virtual ~IClone() = default;
    virtual
    auto clone() -> Clone* = 0;
};

class String:
    public IClone
{
    String(): data_(nullptr){};

    String(const char* data):
        data_(data)
    {};
    String(const String&) = delete;
    
    String(String&& other)
    {
        this->data_ = other.data_;
        other.data_ = nullptr;
    }

    static
    auto create(const char* data) -> Result<String>
    {
        auto* new_data = std::strdup(data);
        return Result<String>::makeOk(String(new_data));
    };

    virtual
    auto clone() -> String* {
        return new String(std::strdup(this->data_));
    };

protected:
    char* data_;
};
```

But this does opt us into having a vtable (which is fine, honestly?). My idea uses the [CRTP pattern](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern).

One other issue: child classes implementing `auto clone() -> Clone*` _must_ return a pointer or reference, and not any other class -- this is because of C++'s rule on overloading functions, which only allow covariance in function return types if they are returning pointers (see [this blog post](https://eli.thegreenplace.net/2018/covariance-and-contravariance-in-subtyping/)).

```cpp
template<typename T>
class MClone
{
    // This is the implicit "shape" of the 
    // implementation function, it may not 
    // actually be defined here
    auto impl_clone() -> std::optional<T&> {};
    auto clone() -> T&
    {
        reutrn static_cast<T*>(this)->impl_clone.value();
    };
};
```

And then implmenetation may look like:

```cpp
class String:
    public MClone<String>
{
public:
protected:

    // needed if we want to call methods protected
    auto impl_clone() -> std::optional<String&> {}
    {
        return make_optional(*new String(this->data_));
    }

    char* data_;
}
```

Right now, the differences between these are fairly small, but the CRTP method gives you a few cool options:

* Since the code isn't instantiated until template expansion occurs, it almost acts as a lazily instantiated way of introspecting a class

* Since your using a (implicitly always safe!) static downcast to the child class, you could make the implementation method virtual if you expect the class to be substyped -- but non-virtual (and thus avoid vtable overhead) if you don't!
    ```cpp
    class SomeClass: public MClone<SomeClass>
    {
        // expected to be overloaded by child classes
        virtual
        auto impl_clone() -> std::optional<SomeClass> = 0;
    };
    ```
* You can something similar to [Rust's associated types](https://doc.rust-lang.org/rust-by-example/generics/assoc_items/types.html) by checking for a child class type:
    ```cpp
    template<typename T>
    class MClone
    {
        // T *must* have a `RetType`
        auto clone_impl() -> T::RetType
        {...};
    };
    ```

* If you're implementation requires any state, you can essentially add the utility added by virtual inheritance without necessarily adding the overhead!  
    For example, an intrusive reference counting type:

    ```cpp
    template<typename T>
    class MRefCounted
    {
        // required impl method to get refcount field
        auto impl_refcount() -> uint32_t& {...};
        // how to release resource once we're finished
        auto impl_release() -> void {...};

        auto increment() -> void {
            auto& val = static_downcast<T*>(this)->impl_refcount();
            val++;
        }
        auto decrement() -> void {
            auto& val = static_downcast<T*>(this)->impl_refcount();
            if (val == 1) {
                val--;
                static_downcast<T*>(this)->impl_release();
            }
            val--;
        }

    };
    ```

* It explicitly annotates at the class level (which is like, the opposite of mixins normally): this means you could add the appropriate static_asserts that some type implements an interface. You could add this as an added trait:
    ```cpp
    #include <type_traits>
    template<typename T>
    struct is_clone
    {
        static constexpr bool value = std::is_base_of<MClone<T>, T>;
    };
    ```

    and as an example of using it:
    ```cpp
    template<typename T>
    auto clone(const T& value) -> Result<T>
    {
        static_assert(
            is_clone<T>::value == true,
            "Type must implement Clone"
        );
        return value.clone();
    }
    ```

## But what about implementing this for other classes?

I don't have a good answer for this, but an option may be something like Abseil's [FTADLE Extension Points](https://abseil.io/tips/218)?

## Is this worth it?

I don't know, probably not! 

A lot of this is already sort of done by the C++ standard library with [concepts](https://en.cppreference.com/w/cpp/keyword/concept), so I mostly just think it's Neat and something you could add to earlier C++ versions.

It's also something I think can show off the utilities of what the CRTP pattern can do. 
