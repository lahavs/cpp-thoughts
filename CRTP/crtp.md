# **Table of contents**
* [Goal](#goal)
* [Minor features of CRTP](#minor_features_of_crtp)
    * [Static polymorphism](#static_polymorphism)
    * [Templated "virtual" functions](#templated_virtual_functions)
* [Major usages for CRTP](#major_usages_for_crtp)
    * [Code reuse](#code_reuse)
    * [Feature injection](#feature_injection)
* [How to think about CRTP](#how_to_think_about_crtp)
* [When to use CRTP](#when_to_use_crtp)


<br></br>
<a name="goal"/>
# Goal
CRTP (Curiously Recurring Template Pattern) it a pattern in C++ that allows to achieve code reuse and static polymorphism.

First, we need to be familiar with the more common alternative - Dynamic polymorphism / Dynamic dispatching.

In dynamic polymorphism, there's a base class with virtual methods.  
A function call is translated to a call to a function table in the `vtable` of the object, given the `vptr`.  
It's called **dynamic** polymorphism, as the actual type (And thus, the actual functions to call) are known only during runtime. 

The alternative to dynamic polymorphism is `static` polymorphism, in which all the types are known during compile time.

As explained in the [Devirtualization documentation](../Devirtualization/devirtualization.md), a virtual function call can be more expensive that a regular function call.

CRTP allows changing virtual functions call to regular ones, and thus have performance advantage when the compiler is unable to devirtualize the function call.

<br></br>
<a name="minor_features_of_crtp"/>
# Minor features of CRTP
I'll start with explaining some minor reasons to use CRTP, just to get them out of the way first.

<br></br>
<a name="static_polymorphism"/>
## **Static polymorphism**
As mentioned, CRTP allows implementing "static polymorphism"

So how static polymorphism is implemented?

The basic structure for a CRTP is a base class (Normally its name ends with "CRTP") that is templated on some type `D`.

Each derived class inherits from the base class, and passes **itself** as the templated type.

When the base class wishes to call a function in the derived class (Similar to calling a regular virtual function), the base class will `static_cast` itself to the type of the derived class (`D`).

For example, the following code (Which uses regular virtual methods):

``` cpp
class Base {
private:
    virtual void foo_impl() = 0;

public:
    virtual ~Base() = default;

    void foo() {
		foo_impl();
    }
};

class Derived : public Base {
private:
    virtual void foo_impl() override {
        // Implementation.
    }
};
```

Can be written to use CRTP like so:
``` cpp
template<typename D>
class BaseCRTP {
public:
    void foo() {
        static_cast<D*>(this)->foo_impl();
    }
};

class Derived : public BaseCRTP<Derived> {
private:
    friend BaseCRTP<Derived>; // So 'BaseCRTP<Derived>' could access the private 'foo_impl()'

    void foo_impl() {
        // Implementation.
    }
};
```

Since the compiler knows at **compile time** the type _D_ when it compiles `BaseCRTP<D>::foo()`, which in turn calls `D::foo_impl()`, then the compiler can easily _inline_ `D::foo_impl()` into `foo()`.

The reason why CRTP is useful here is because the compiler will **most likely not** be able to devirtualize the virtual call to `foo_impl()` in `Base::foo()`.

As for static polymorphism using CRTP, one could write a function that accepts a `BaseCRTP<D>` that will be useable by any class that inherits from that class:
``` cpp
template<typename D>
void func(BaseCRTP<D>& obj) {
    obj.foo();
}
```
In contrast with a regular:
``` cpp
template<typename T>
void func(T& obj) {
    obj.foo();
}
```

The first option only allows passing objects that derive from `BaseCRTP`, and disallow passing other arbitrary objects (That might define their own `foo()` function).  
The former offers better type safety (Disallows accidentally passing "wrong" objects).  

While the latter is possible, I haven't seen any use for such a construct, and I don't think this is the "correct" way to think about CRTP (Although I think it's still important to be familiar with it).

<br></br>
<a name="templated_virtual_functions"/>
## **Templated "virtual" functions**
Another interesting property of CRTP is that unlike regular dynamic polymorphism where virtual functions can't be templated (since the compiler, when it sees call to the virtual function, will not know how to instantiate the function at the derived class), with CRTP it's possible (Since in CRTP everything is known at compile-time).  

For example:
``` cpp
template<typename D>
class BaseLoggerCRTP {
public:
    template<typename T>
    void log(const T& obj) {
        std::cout << "Starting logging.." << std::endl;
        static_cast<D*>(this)->log_impl(obj);
        std::cout << "Finished logging.." << std::endl;
    }
};

class CoutLogger : public BaseLoggerCRTP<CoutLogger> {
private:
    friend BaseLoggerCRTP<CoutLogger>;

    // Templated!
    template<typename T>
    void log_impl(const T& t) {
        std::cout << t << std::endl;
    }
};
```

<br></br>
<a name="major_usages_for_crtp"/>
# Major usages for CRTP
There are 2, closely related, major usages for CRTP: **Code reuse** and **"Feature injection"**.


<br></br>
<a name="code_reuse"/>
## **Code reuse**
At its core, CRTP allows calling a derived class' methods from within the base class, similarly to using a virtual function.

Therefore, this pattern is closely related to the [Template design pattern](../Template%20Design%20Pattern/template_design_pattern.md).

We could say that CRTP is a technique that enables implementing the "Template design pattern" **statically**, using inheritance.

For example:
``` cpp
template<typename D>
class BaseCRTP {
public:
    void send() {
        static_cast<D*>(this)->pre_send();
        // Do actual sending
        static_cast<D*>(this)->post_send();
    }
};

class Derived1 : public BaseCRTP<Derived1> {
private:
    friend BaseCRTP<Derived1>;

    void pre_send() {
        // Implementation 1
    }

    void post_send() {
        // Implementation 1
    }
};

class Derived2 : public BaseCRTP<Derived2> {
private:
    friend BaseCRTP<Derived2>;

    void pre_send() {
        // Implementation 2
    }

    void post_send() {
        // Implementation 2
    }
};
```

As one can see, the `BaseCRTP` class encapsulates the "template" of the logic, while each derived class only implements its own unique logic, while everything it "wired" in **compile time**.

This is one of the main usage for CRTP that I'm aware of.

<br></br>
<a name="feature_injection"/>
## **Feature injection**
With CRTP, one can also extend the functionally a class with the features of another class.

While this is similar to the "Template method pattern", the goal is a bit different.

For example, let's consider implementing an iterator class for a new object.

There are multiple operations that we want to perform on an iterator:
1. Pre/Post increment/decrement
2. operator[], operator->
3. Comparison operators (==, !=, <, >, <=, >=)
4. Arithmetic operators (+, -, +=, -=)
5. Dereference (operator*)

Most of the logic is almost identical between all the different iterators.

_Boost_ has a `iterator_facade` class, that encapsulates the boilerplate, and only requires a sub-set of operations to be defined from the user's class.

Using it is quite easy:

``` cpp
class MyIterator :
    public boost::iterator_facade<MyIterator,                        /** CRTP!                                     */
                                  const char,                        /** Value type (Return value for dereference) */
                                  std::random_access_iterator_tag> { /** Iterator tag                              */
private:
    const char* m_ptr;

public:
    explicit MyIterator(const char* ptr)
     :  m_ptr(ptr)
    { }

    const char& dereference() const {
        return *m_ptr;
    }

    bool equal(const MyIterator& rhs) const {
        return m_ptr == rhs.m_ptr;
    }

    void increment() {
        ++m_ptr;
    }

    void decrement() {
        --m_ptr;
    }

    void advance(ptrdiff_t n) {
        m_ptr += n;
    }

    ptrdiff_t distance_to(const MyIterator& iter) const {
        return iter.m_ptr - m_ptr;
    }
};
```

Notice that only a few functions need to be implemented! With this, we can do:
``` cpp
const char* str = "123";
MyIterator i(str), j(str);
++i;
std::cout << *i << std::endl;
std::cout << j[1] << std::endl;
std::cout << std::boolalpha << (i >= j) << std::endl;
std::cout << std::boolalpha << (i == j) << std::endl;
```

The basic idea behind how `iterator_facade` is implemented is quite easy:

Let's consider the comparison operators:
* operator<(i, j) is implemented as:  
    `return 0 > -i.distance_to(j)`  

* operator>(i, j) is implemented as:  
    `return 0 < -i.distance_to(j)`

And so on..

<br></br>
<a name="how_to_think_about_crtp"/>
# How to think about CRTP
Similarly to the "Template design pattern", it's important not to think about CRTP's inheritance as a "is-a" relationship (Unless static polymorphism is needed, but this is rarely the case).

Instead, think about it as simply a technique that enables code reuse without paying the cost of dynamic dispatching.

<br></br>
<a name="when_to_use_crtp"/>
# When to use CRTP
I think it's safe to say that a "dynamic" Template design pattern (i.e. Using virtual methods) is much more readable and maintainable than using CRTP.

This really should be the top priority when writing code - If the code is unnecessary complicated or "clever", then unless there's a very good reason for it, the code often should be "dumbed down".

As always, a major reason to not write simple and straightforward code, is **performance**.

As stated above, the only main advantage that CRTP brings is the removal of a virtual function.  
Therefore, if the rest of the code is either:
1. Not a bottleneck (The rest of the code is much slower than a virtual call).
2. Not performance critical (Not in the "hot path" of data).

Then there's really no need to use CRTP in the first place. Just use simple inheritance with regular virtual functions.

And to reiterate, CRTP is not a technique that can be used everywhere: It can be used only when the concrete type is known at **compile time**.

In cases where the static type of the object it not known, then regular dynamic polymorphism must be used.
