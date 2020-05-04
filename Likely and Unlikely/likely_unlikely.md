# **Table of contents**
* [Goal](#goal)
* [The likely/unlikely macros](#the_likely_unlikely_macros)
* [Example](#example)
* [When to use](#when_to_use)

<br></br>
<a name="goal"/>
# Goal
These are macros that enable telling the compiler specific information about the expected value of an expression, which can help the compiler optimize the code in ways that it can't do (Without this extra information).  

Given a branch:
``` cpp
foo1();
if (cond) {
    foo2();
}
foo3();
```

The compiler can order the functions calls in the generated assembly as either:
1. `foo1()`, `foo2()`, `foo3()`
2. `foo1()`, `foo3()`, `foo2()`

i.e. First have `foo1()`, then the block for `foo2()`, and then the block for `foo3()` - Or after `foo1()` have the block for `foo3()` and afterwards `foo2()`  

The order can have an effect on performance:  
If the compiler chose the first option, and the condition is rarely met, then whenever the CPU will process this code it will need to "jump" over the code for `foo2()` (As the code regarding `foo3()` is physically located after the chunk of code regarding `foo2()`).  

While it's true that the CPU's branch predictor will quickly learn that the condition is mostly never false, (And thus speculatively execute the code for `foo3()` even before the condition `cond` is fully evaluated), there's still the issue of "jumping" to the `foo3()` code:  
If the code for `foo2()` is large, then the code for `foo1()` and `foo3()` might be in different cache lines in the I-cache, meaning there's most likely going to be a I-cache miss before executing `foo3()`!  
<br></br>
If instead, the code was ordered the 2nd way, then the code for `foo3()` immediately follows `foo1()`, meaning they're more likely to in the same cache line.  

Most modern compilers have sophisticated heuristics to determine which branch is more likely: For example, GCC treats comparisons of pointers against NULL as being unlikely to happen (i.e. Pointers are rarely NULL, when they're compared against NULL).  
Meaning in the following code:
``` cpp
foo1();
if (NULL == ptr) {
    foo2();
}
foo3();
```
The body of `foo2()` will be far away from the rest of the code (i.e. The compiler will choose option #2)

<br></br>
<a name="the_likely_unlikely_macros"/>
# The likely/unlikely macros
GCC (And other compilers) export a way to let the programmer tell the compiler explicitly which branch is more likely, using the `__builtin_expect()` builtin:

`__builtin_expect(exp, val);`  

Where exp is an expression to evaluate, and val is the expected value.  

This builtin function expresses the compiler the expect value of the expression, which can help him produce more efficient code.  

This builtin can help the programmer tell the compiler which branch is more likely to occur, which will make the compiler reorder to code in such a way that favours the expect branch, i.e. put it physically close to the code before that branch. Code in the "unlikely" branch is usually placed at the **very end** of the function.  

Most of the time we deal with booleans, in which the following "syntactic" macros are used:  
``` cpp
#define likely(x)   __builtin_expect((x), true)
#define unlikely(x) __builtin_expect((x), false)
```

Using them is quite easy:
``` cpp
foo1();
if (unlikely(cond)) {
    foo2();
}
foo3();
```

<br></br>
<a name="example"/>
# Example
Let's look at the code:
``` cpp
void func(bool a) {
    if (a) {
        foo1();
        foo2();
        foo1();
        foo2();
        foo1();
        foo2();
        foo1();
        foo2();
    }

    puts("Writing..\n");
    foo1();
}
```

The generated assembly is:
``` asm
   4009e0:  sub    $0x8,%rsp

   ; if (a) {
   4009e4:  test   %dil,%dil
V- 4009e7:  je     400a11 <func(bool)+0x31>
V  4009e9:  callq  400ad0 <foo1()>
V  4009ee:  callq  400ae0 <foo2()>
V  4009f3:  callq  400ad0 <foo1()>
V  4009f8:  callq  400ae0 <foo2()>
V  4009fd:  callq  400ad0 <foo1()>
V  400a02:  callq  400ae0 <foo2()>
V  400a07:  callq  400ad0 <foo1()>
V  400a0c:  callq  400ae0 <foo2()>
V  ; }
V
V  ; puts("Writing..\n");
>  400a11:  mov    $0x400b94,%edi
   400a16:  callq  400810 <puts@plt>
   400a1b:  add    $0x8,%rsp

   ; foo1();
   400a1f:  jmpq   400ad0 <foo1()>
```

As one can see, the generated assembly closely follows the code itself (First the _if_ statement, then the body of the _if_ statement, and then the code afterwards).  

Now, let's change the condition to:
``` cpp
if (unlikely(a)) {
```

The generated code is:
``` asm
   4009e0:   48 83 ec 08             sub    $0x8,%rsp

   ; if (unlikely(a)) {
   4009e4:   40 84 ff                test   %dil,%dil

   ; puts + foo1()
   4009e7:   75 17                   jne    400a00 <func(bool)+0x20>      >
>  4009e9:   bf 94 0b 40 00          mov    $0x400b94,%edi                V
^  4009ee:   e8 1d fe ff ff          callq  400810 <puts@plt>             V
^  4009f3:   48 83 c4 08             add    $0x8,%rsp                     V   If 'a' is true
^  4009f7:   e9 d4 00 00 00          jmpq   400ad0 <foo1()>               V
^  4009fc:   0f 1f 40 00             nopl   0x0(%rax)                     V
^                                                                         V
^  ; 'if' body:                                                           V
^  400a00:   e8 cb 00 00 00          callq  400ad0 <foo1()>              <-
^  400a05:   e8 d6 00 00 00          callq  400ae0 <foo2()>
^  400a0a:   e8 c1 00 00 00          callq  400ad0 <foo1()>
^  400a0f:   e8 cc 00 00 00          callq  400ae0 <foo2()>
^  400a14:   e8 b7 00 00 00          callq  400ad0 <foo1()>
^  400a19:   e8 c2 00 00 00          callq  400ae0 <foo2()>
^  400a1e:   e8 ad 00 00 00          callq  400ad0 <foo1()>
^  400a23:   e8 b8 00 00 00          callq  400ae0 <foo2()>
^- 400a28:   eb bf                   jmp    4009e9 <func(bool)+0x9>
```

Notice that in this case, since it's **unlikely** to enter the if statement, its body is moved to the **end** of the function.  
As we can see, the "likely" case (Where `a` is false) is following immediately after the condition.  
This means that the CPU will have the next instructions ready in the I-cache, which can be a big win if the code is a bottleneck.  

If the "unlikely" case happens (`a` is true) then one can see the how the instruction pointer will "jump around" the code: From the branch in address 0x4009e7 it will jump all the way **"down"** to execute the if body, and then jump **"up"** to execute the code that follows the _if_ statement.

<br></br>
<a name="when_to_use"/>
# When to use
If the prediction on the value doesn't match the actual value, then using `__builtin_expect()` can actually **hurt** performance.  

Therefore, before using these macros, one should be absolutely certain that the expected value is indeed the value that will show up when the code is ran.  
If the value only sometimes matches the expected value, then do not use `__builtin_expect()`.  

Some places where using `__builtin_expect()` is appropriate is when dealing with errors:  
Assuming error conditions are rare, and not part of the "hot path", then deciding whether or not to branch into the error handling can be accompanied with the `unlikely()` macro.  

For example:
``` cpp
if (unlikely(error_condition)) {
    // Log + Return an error
}
```

But still, even if the expected value matches **all the times** with the actual value, the performance will not be improved **greatly** (Of course it depends on the actual code, but most of the times it will not have any noticeable effect on the code).  

Therefore, in my opinion, none of us should really use these macros without first running a benchmark, since even if the actual value of the expression matches 100% of the times with the expected value, the extra notation of using `likely()`/`unlikely()` simply clobbers the code and makes it slightly harder to read, especially when it's used frequently in a single function.  

I believe that **readability and maintainability** are the most important characteristics of code. Therefore, if something makes one of them harder without a **real** reason for it, one should shy away from constructs that make them harder.
