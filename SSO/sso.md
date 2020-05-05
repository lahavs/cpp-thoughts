# **Table of contents**
* [Preface](#preface)
* [SSO](#sso)
* [SSO in the wild](#sso_in_the_wild)
* [Implementation](#implementation)
* [When to use SSO](#when_to_use_sso)
* [When NOT to use SSO](#when_not_to_use_sso)

<br></br>
<a name="preface"/>
# Preface
SSO (Small string optimization), also called "Small object optimization" and "Small buffer optimization", is an optimization on how to layout array-like objects in memory, aimed to improve the performance when manipulating such objects.
<br></br>

As you already know, constructing an array with static size can be done very quickly: By allocating it on the stack, allocating and deallocation are a simple matter of moving the `%rsp` pointer.  
But it's not always possible to declare an array on the stack:
1. Sometimes the size of the array is not known at compile time. [(:star:)](#star_1)
2. By default the stack is rather small compared to the heap, thus care must be taken to not reach its capacity.

<br></br>
<a name="star_1"/>
(:star:)
> Note that I'm deliberately not talking about:  
> 1. `alloca()` (A function similar to `malloc()`, which allocates an array of dynamic size on the **stack** instead of the heap):
>     ``` cpp
>     void func(const size_t size) {
>         void* buf = alloca(size);
>     }
>     ```
>
> 2. Or VLAs (Variable Lengthed Arrays - An extension allowing declaring array on the stack with a **dynamic size**):
>     ``` cpp
>     void func(const size_t size) {
>         char buf[size];
>     }
>     ```
>
> These 2 options, while allowing creating arrays of dynamic sizes on the stack, should **never** be used in a code that attempts being secure and portable, since:  
> 1. Error reporting for `alloca()` is non standard (Will probably never return NULL, but instead will happily return pointers until the page guard at the end of the stack (If one exists) is (hopefully) returned (And only when it's **dereferenced** will an error occur)).
> 2. It's very easy to overflow the stack using them:  
> For example with `alloca()`, it's possible that with a large enough size it will "jump" over any page guard (Again, if ones exist) and start returning pointers which are not on the stack.
> 3. Neither `alloca()` or VLAs can be used for a data member of a class, which is more often than not what we want (Define a data structure that exports some API, while internally being implemented with an array)


Thus, in cases where the size of the array is not known, one must use heap allocations in order to store the buffer.  
This alone, depending on the situation, can be a bottleneck in the application.

The reason for this is twofold:
1. The simple act of calling `malloc()` might be enough to hurt the performance.
2. Objects that internally dynamically allocate arrays are doing so with incremental sizes. [(:star:)](#star_2)  
    Each such incremental increase in the size requires:
    1. Allocating a new buffer with the new size.
    2. Constructing the elements in the new buffer, using the elements from the old buffer.
    3. Destroying the objects in the old buffer.
    4. Freeing the memory of the old buffer.  


The 2nd thing can hurt performance when the initial size of the array is **small**: When the array goes through these "small allocations" steps, the object performs a lot of actions that is normally doesn't really need to be doing, and thus can hurt performance.

<br></br>
<a name="star_2"/>
(:star:)
> In most implementations, the "growth policy" (i.e. Deciding the size of the new buffer) is usually **exponential**.  
> Exponential growth and not, say, linear growth, gives the best runtime performance, as it requires less memory allocations when the buffer continuously grows.
>
> For example, in libstdc++ both `std::vector` and `std::string` use exponential growth with base **2** (e.g. Starting from an empty vector, the sizes will be: 1, 2, 4, 8, 16, ...)


Depending on the class being used, it's possible to avoid these "small allocations". For example, consider the code:
``` cpp
void func(const size_t size) {
    std::vector<uint8_t> vec;

    for (size_t i = 0; i < size; ++i) {
        vec.push_back(get_value());
    }
}
```

Instead of allowing the vector to incrementally allocate its buffer, we can just reserve enough space at the beginning of the execution:
``` cpp
void func(const size_t size) {
    std::vector<uint8_t> vec;
    vec.reserve(size);

    for (size_t i = 0; i < size; ++i) {
        vec.push_back(get_value());
    }
}
```

This alone can tremendously improve performance. But sometimes, even this is not enough, and the **act of dynamically allocating memory** is enough to greatly hurt performance.

<br></br>
<a name="sso"/>
# SSO
SSO aims to eliminate the need for having **any** dynamic memory allocation for the **common cases**.

At its core, SSO is a hybrid approach between using an array on the stack and using an array allocated on the heap:

1. A small portion of the object is set to be an array with a static size. This array is often called the "local" or "inline" array.
2. If only a small number of elements are present in the object, they will all be placed in the local buffer.
3. If a large number of elements are placed into the object, then the object will "transition" (Often called "mutate" or "grow") into using a dynamically allocated array (Called the "non-local buffer", or the "dynamic buffer"), with a size proportional to the size of the local buffer (e.g. Start by 2x the size of the local buffer, depending on the growth policy for the dynamically allocated buffer).

When the array is using the "local" buffer it is said to be in "local mode", otherwise it is said to be in "non-local mode", or "dynamic mode".

Thus, SSO solves both of the aforementioned problems with dynamic memory allocations:
1. If the number of elements in the array does not exceed the size of the local buffer, no dynamic memory allocations will take place **at all**.
2. Even if the array mutates to a non-local buffer, no "small allocations" will take place, since the size of the non-local buffer starts with the size of the local buffer - The object effectively "skipped" the "small allocations" step.


Of course SSO does not come without any drawbacks:
1. If the size of the local buffer is much larger than the number of elements that will actually be present in the object in any point of time, then the existence of this unused space can hurt the spatial locality of the code.
2. Implementing an SSO object includes making trade-offs, which can hurt performance if the wrong ones are taken.  
    For example, the code that manages the local/non-local modes can have a negative impact on performance, as a naive implementation might add an **extra** branch in the "add new element" functions, as they will first **check if in local mode**, and if not check if reached the current capacity.  
    This extra branch (Which is executed whenever a new object is added to the array) might hurt performance.
3. Move operations on an SSO-ed object in local mode might be less efficient than an implementation which only uses dynamic memory allocations, as moves on the local buffer require moving **each element** into its new location, in contrast with a pure dynamically allocated buffer, in which only a simple **pointer move** is needed.


<br></br>
<a name="sso_in_the_wild"/>
# SSO in the wild
Most standard library implementations nowadays implement `std::string` using SSO.

For example, following GCC 5, libstdc++'s `std::string` is implemented using SSO. Beforehand, `std::string` was implemented using copy-on-write semantics using reference counting.

One can compile with COW strings by compiling with `-D_GLIBCXX_USE_CXX11_ABI=0` (But it affects other classes as well, though).

Under libstdc++, `std::string` uses a 15 characters-long local buffer, meaning its total size is 16 bytes (An extra byte for the NULL terminator).

Sadly, due to restrictions from the standard, `std::vector` **cannot** be implemented with SSO.


<br></br>
<a name="implementation"/>
# Implementation
Here is a very simple (And incomplete) implementation of a `SmallVector`, yet it shows some important design choices used in the real world (Requires C++17):
``` cpp
template<typename T, size_t N>
class SmallVector {
private:
    T*     m_begin;
    size_t m_len;
    
    union {
        size_t m_capacity;
        T      m_local_arr[N];
    };

public:
    SmallVector()
     : m_begin(m_local_arr), m_len(0)
    { }

    ~SmallVector() {
        // Call the destructor on the existing elements
        std::destroy_n(m_begin, m_len);

        if (! is_local()) {
            free(m_begin);
        }
    }

    size_t capacity() const {
        return is_local() ? N : m_capacity;
    }

    bool is_local() const {
        return (m_begin == m_local_arr);
    }

    /**
     * Mutate the vector:
     * 1) Point 'm_begin' to a new dynamically allocated buffer.
     *    The size of the new buffer will be larger that the capacity usually with exponential growth.
     * 2) Set 'm_capacity' to the size of the new buffer.
     */
    void mutate() {
        // We use exponential growth policy with base 2.
        const size_t new_capacity = 2 * capacity();
    
        // Allocate the new dynamic buffer
        T* new_data = (T*)memalign(alignof(T), sizeof(T) * new_capacity);
        if (! new_data) { throw ... }

        // Move the elements from the previous buffer to the new one
        std::uninitialized_move_n(m_begin, m_len, new_data);
    
        // Destruct the left-over elements
        std::destroy_n(m_begin, m_len);
    
        // Free the previous buffer (If it was dynamic)
        if (! is_local()) {
            free(m_begin);
        }

        m_begin    = new_data;
        m_capacity = new_capacity;
    }

    void push_back(const T& val) {
        if (capacity() == m_len) {
            mutate();
        }
    
        // Construct the element in the matching place with 'val'
        ::new(m_begin + m_len) T(val);
        ++m_len;
    }

    void pop_back() {
        if (0 == m_len) { throw ... }
        --m_len;
        m_begin[m_len].~T();
    }

    auto begin() {
        return m_begin;
    }

    auto end() {
        return m_begin + m_len;
    }
};
```

Notice that this implementation involves some interesting optimizations and design choices:
1. `m_begin` is used to point at the "active" buffer (Either `m_local_arr` if in local mode, or the dynamically allocated buffer when in non-local mode).  
    This simplifies the code, as it removes the need to branch (Depending on whether or not it's in local mode) on almost all of the methods in order to get the current pointer to the "active buffer".  
    These methods include `push_back()`, `pop_back()`, `begin()`, `end()` etc.  
    As these is the most commonly used methods on a vector, it makes sense to optimize their code.  

2. Because `m_begin` points to the current "active buffer", no extra "is_local" boolean is added - Instead this choice is made by checking if `m_begin` points at the `m_local_arr`.  
    Indeed this means that checking if the vector is in local mode is slightly slower due to the branch (Compared to just use a dedicated boolean for it), but notice that this is not checked very often:  
    In the presented methods above, only `push_back()` checks if the vector is local mode, and it does so only when it **reached its capacity**.  
    But! Since an exponential growth policy is used, the vector will often not reach its full capacity (And thus need to re-allocate).  

3. When the vector mutates to a dynamic buffer, a new buffer is dynamically allocated, and all of the existing elements in the previous buffer are moved to the new buffer.  
    This might seem wasteful at first, as it looks reasonable to keep using the local buffer (i.e. Part of the vector will be stored in the local buffer, and the rest in the dynamic buffer).  
    This will remove the need to move the elements upon a mutation, and will remove the size of the dynamically allocated buffer.  
    But still, whenever the vector is in non-local mode the local buffer is **not used**.  
    The reason is quite simple: Implementing like so could potentially hurt performance, which mainly come from the fact that the vector's data is no longer **continuous in memory**:
    1. It suppresses other optimizations, for example #5 below
    2. Accessing a random element in the vector will be a bit slower, as it will branch on whether or not the vector is in local mode.
    3. Also, iterators will no longer be able to be just raw pointers, as its `operator++` will need to have logic that know when to "jump" from the local buffer into the dynamic buffer. This will also mean that iteration over the vector will be slower.
    4. And actually more importantly, vectors are usually assumed to be continuous in memory. Therefore, in such an implementation, properly implement the `data()` method is impossible.  

4. Due to #3 optimization, we get that `m_capacity` and `m_local_arr` are never used at the same time:  
    If the vector is in local mode, then the capacity is simply `N`. Otherwise, we do need to store the capacity, but there's no need for `m_local_arr` anymore.  
    Therefore, in order to save **space of the object itself**, they are placed in an union.  
    While this will slow down `capacity()` a bit due to branching depending on if it's in local mode or not, `capacity()` is rarely called on a vector, therefore it's something we will not really pay for.  

5. Even without #4 optimization, there's an extra reason for `m_local_arr` to still be present inside a union:  
    If it was just a regular data member, then the constructor of `SmallVector` would have **default initialized** ALL of the members in the local buffer (This is mandatory by the standard).  
    This is of course wasteful - We don't want to initialize elements that we might not even use. Also, it's probably more efficient (Depending on the type `T`) to directly initialize each element with the correct arguments (e.g. Like in `push_back()`), instead of default initializing and then assigning them to a new value.  
    By simply moving it inside a union, `m_local_arr`'s elements are no longer initialized in the constructor.  
    Note that `SmallVector`'s constructor doesn't initialize `m_local_arr`. If it were, then the elements will again be default-initialized.  

<br></br>
Of course these are just some specific optimizations - Other implementations may add their own optimizations, as they have different trade-offs to consider.

For example, let's consider some of LLVM's `SmallVector` implementation details:
1. The implementation allows to [type-erase](../shared_ptr) the **templated size** of the local buffer, something that is not possible with our implementation.  
    This is done by making `SmallVector` inherit from a `SmallVectorImpl` class which is not templated on the size but rather stores it inside its `m_capacity` data member, and the code that wants to use a type-erased `SmallVector` simply uses `SmallVectorImpl`.
    Basically something like:
    ``` cpp
    template<typename T>
    class SmallVectorImpl {
    private:
        size_t m_len;
        size_t m_capacity;
        T*     m_begin;
    
    public:
        SmallVectorImpl(T* buf, size_t capacity)
         :  m_len(0),
            m_capacity(capacity),
            m_begin(buf)
        { }
    
        /** push_back(), pop_back() ... */
    };
    
    template<typename T, size_t N>
    class SmallVector : public SmallVectorImpl<T> {
    private:
        union {
            T m_local_arr[N];
        };
    
    public:
        SmallVector()
         :  SmallVectorImpl<T>(m_local_arr, N)
        { }
    };
    ```
    And now one can have a function that accepts `SmallVectorImpl`:
    ``` cpp
    void func(SmallVectorImpl<int>& vec) {
        vec.push_back(1);
    }
    
    SmallVector<int, 5> vec1;
    func(vec1);
    
    SmallVector<int, 3> vec2;
    func(vec2);
    ```

    Thus the logic that works with a `SmallVector` is not bound by the capacity of the local buffer.

    While this type-erasure has its usages, this suppresses our #4 optimization: In which `m_capacity` and `m_local_arr` are stored in a union, and thus requires less space.  
2. Their `m_len` and `m_capacity` are stored as a 32-bit unsigned integer. This has the effect of reducing the size of each `SmallVector` instance by a total of 8 bytes (On 64bit machines, compared to the usual case in which `size_t` will normally be used), at the expense that the implementation doesn't support vectors containing more than 4G of elements.

<br></br>
<a name="when_to_use_sso"/>
# When to use SSO
Only if the **expected** number of elements is rather **small**, SSO could be a good optimization to try.

And as it always goes with optimizations: One should be using SSO only if he profiled the code and verified that indeed the **memory allocations** are a bottleneck.

<br></br>
<a name="when_not_to_use_sso"/>
# When NOT to use SSO
Sometimes, it's not even needed to use SSO:
* If an upper limit on the size is known, it might be best to just define an array on the stack with that maximum value as its size.  
    The usual drawback with this approach is that it might waste a lot of memory in case the limit is technically very large, and is rarely reached.

Some other times, it might not be beneficial:
* If in the majority of cases the number of elements that will be present in the array is rather **large**, then using SSO might not be beneficial, or even possible (Due to limited stack space).
