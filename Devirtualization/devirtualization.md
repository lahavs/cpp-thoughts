# **Table of contents**
* [Goal](#goal)
* [Implementation](#implementation)
* [Examples](#examples)
* [How to help the compiler](#how_to_help_the_compiler)
* [Speculative devirtualization](#speculative_devirtualization)
* [When to use the final keyword](#when_to_use_the_final_keyword)

<br></br>
<a name="goal"/>
# Goal
While virtual functions are great, they have their drawback. Here we'll talk about their performance issues:
1. The extra `vptr` in each object increases the size of the object by 8 bytes (sizeof pointer).  
   This can have a great impact in case the objects are small: If the object is empty, then only ~64/8 = 8 such objects can reside in a single cache line.  
   On the other hand, if static polymorphism was used, each object would take up only 1 byte, meaning now ~64/1 = 64 objects can reside in a single cache line.  
   This can have an impact if we iterate over a large number of objects which are consecutive in memory.  
   I usually don't see these kind of issues (There are other major parts of the code that are much less performant compared to this), but it's still important to know about it.
2. Each virtual function calls costs an extra indirections:
   1. Dereferencing the this pointer to get the `vptr`.
   2. Jumping to `vptr + x`, where x is the location of the function in the `vtable`.
   3. Dereferencing `vptr + x` to get the actual function address.
3. The biggest issue with dynamic polymorphism is that the function to call can't be **inlined** (Usually, as we'll now see).

Devirtualization is an **optimization** made by the compiler to **remove these overheads** when calling a virtual function.  

<br></br>
<a name="implementation"/>
# Implementation
The idea behind devirtualization is that if the compiler can prove that at a given virtual function call site, only 1 function is a candidate for being the target of the dynamic dispatch, then the compiler will convert the virtual function call to a regular function call.

<br></br>
<a name="examples"/>
# Examples
Let's look at the code:
``` cpp
class I {
public:
    virtual ~I() {}
    virtual int func() = 0;
};

class A : public I {
public:
    virtual int func() override {
        return 3;
    }
};

int foo(A& a) {
    return a.func();
}
```

The generated assembly for `foo()` will look something like:
``` asm
mov    (%rdi),%rax
mov    0x10(%rax),%rax
jmpq   *%rax
```

Here `%rdi = &a` (The first parameter to `foo()`).  
The generated assembly does:
1) Reads the `vptr` (The first member in class `A`) into `%rax`.
2) Adjust `%rax` to point at the 3rd pointer in the `vtable` (`A::func()` in this case).
3) `%rax` is dereferenced and `A::func()` is called.

<br></br>
<a name="how_to_help_the_compiler"/>
# How to help the compiler
C++11 added the `final` keyword, which can be used at both a class and method declaration.
* If a class is marked `final`, then no class can inherit from this class.
* If a method is marked `final`, then no class that derives from that class can override this method.

For example:
``` cpp
class Interface {
public:
    virtual void func() = 0;
};

class Derived1 : public Interface {
public:
    virtual void func() final override;
};

class Derived2 final : public Interface {
public:
    virtual void func() override;
};
```

There are 2 reasons for the `final` keyword:
1) Express in the interface that either the class should not be derived from, or the method shouldn't be overridden.
2) Help the compiler devirtualize calls :)

Going back to the first example, by marking either `A` or `func()` as `final` the generated assembly becomes:
``` asm
mov    $0x3,%eax
retq
```

Since the compiler know that `A::func()` is the only candidate to be called, it immediately calls it (And inlined it as well).

<br></br>
<a name="speculative_devirtualization"/>
# Speculative devirtualization
An interesting optimization made by the compiler that the compiler tries to **guess** which function will be called in a given virtual function call site.  
Normally this is done by comparing the virtual function pointer (Obtained from the `vtable`), and compare it with the actual address of the function the compiler guess it points to.

This optimization is mostly used to enable **inlining** the target function inside the caller (Something that is not possible with virtual functions).  

For example, the generated assembly for the first example can look like (A bit tidied up):
``` asm
   ; Load the pointer to the virtual function
   mov    (%rdi),%rax
   mov    0x10(%rax),%rax
   
   ; Check if the virtual function pointer points at `A::func()`
   ; If so, directly call it here. If not, do an indirect call via *%rax
   cmp    $0x400960,%rax
V- jne    4009a8 <foo(A&)+0x18>
V  mov    $0x3,%eax
V  retq
>  jmpq   *%rax
```

If the function was called with an object that inherits from `A` and overrides `A::func()`, then it will have a small performance penalty due to the comparison (Although the branch predictor will quickly catch up, if it's mostly called like so).  

GCC uses sophisticated heuristics to determine which function call, if any, is likely to be the target of the virtual call.  
For example, it seems that if it detects that in the current translation unit no other class inherits from A, then it will guess that no other class will inherit from A in the **entire program**, and thus optimize for this case.  

Theoretically, it can also guess more than 1 targets (And compare against say 2 pointers), though I've never seen this happen.

Let's look at a more complex code:
``` cpp
int foo(A& a) {
    a.func();
    puts("123\n");
    return 0;
}
```

The generated assembly will look something like (Removed some unnecessary lines):
``` asm
   4009d0:  sub    $0x8,%rsp
   
   ; %rax = The pointer to 'A::func()' in the vtable
   4009d4:  mov    (%rdi),%rax
   4009d7:  mov    0x10(%rax),%rax
   
   ; if (A::func == %rax)
   4009db:  cmp    $0x4009a0,%rax
   4009e1:  jne    4009f8 <foo(A&)+0x28>
   
   ; puts("123\n");
>  4009e3:  mov    $0x400b10,%edi
^  4009e8:  callq  400780 <puts@plt>
^  4009ed:  xor    %eax,%eax
^  4009ef:  add    $0x8,%rsp
^  4009f3:  retq
^  4009f4:  nopl   0x0(%rax)
^
^  ; Indirect call to *%rax
^  4009f8:  callq  *%rax
^- 4009fa:  jmp    4009e3 <foo(A&)+0x13>
```

The compiler again guess that `a.func()` will call `A::func()` (0x4009e1). In this case it also inlined and removed it completely, as it noticed that the function does nothing since its return value is ignored.  

Do you notice something interesting here? If the compiler guessed poorly (The virtual function will not call `A::func()`), then the code that handles it placed at the very end of the function.  

If you read the [Likely/unlikely documentation](../Likely%20and%20Unlikely/likely_unlikely.md) this will look familiar:  
The compiler is **very confident**  in his guess!. It's as if the code was:
``` cpp
if (likely(A::func == %rax))
```
That means calling this `func()` function with objects other than A (That override the `A::func()` method) will have a performance penalty, for all the reasons described in the "Likely/unlikely" documentation.

<br></br>
<a name="when_to_use_the_final_keyword"/>
# When to use the `final` keyword
The drawback with using `final` is that if in the future we might want to inherit from a final class / override a final method, we will not be able to.  
In an internal code base this drawback may not really exist, as we can simply remove the 'final' keyword without breaking changing API with some other project we're integrated with.

Therefore, it's generally best to use these keywords whenever possible. For this we use the following GCC warning:

`-Wsuggest-final-types`

With this warning, GCC will warn whenever a virtual function call site was devirtualizable, only if the class at hand was marked as `final`.  
This will help pinpoint the exact classes that should be marked as final.  

<br></br>
A few final notes:
1. GCC is not perfect, and sometimes warns about classes that should be marked as `final`, but are there's a class that actually derives from this class.  
This usually happens due to `#ifdef` in the code (e.g. Only under some `#ifdef` there's a class that inherits from the class at hand), but can also show up due to the fact that a compiler compiles each translation unit by itself (And thus it's not aware of class in other translation units that inherit from that class).  
LTO (Link Time Optimizations) somewhat helps with removing some of these false-positives, but still some might pop up.  
In these cases simply ignore this warning when declaring the class:
    ``` cpp
    #pragma GCC diagnostic push
    #pragma GCC diagnostic ignored "-Wsuggest-final-types"
    class Derive : public Base {
        // ...
    };
    #pragma GCC diagnostic pop
    ```

2. GCC also supports the `-Wsuggest-final-methods`.  
This warning is similar to `-Wsuggest-final-types`, except this is for methods, and not classes.  
In my code this warning is currently **NOT** in use - The number of false-positives we will get is **HUGE**. Having some `#ifdef` around a few classes is bad as it is, but having these `#ifdef` around a large number of methods will just bloat the code and make it unreadable.  
Also, these 2 warnings don't go too well together. For example:
    ``` cpp
    void foo(Derive& d) {
        d.func();
    }
    ```
    In this virtual function call site, the compiler will warn about BOTH the class `Derive` being able to be marked as final and also the method `func()`, which can bombard the output of the compiler..  
    
    Also, marking each and every method as final can quickly reduce the readability of the code.  
    Having it only for a few class definitions is considered not THAT much of a bad thing.
