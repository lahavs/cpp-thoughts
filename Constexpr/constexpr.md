# **Table of contents**
* [Goal](#goal)
* [Terminology](#terminology)
* [Basic rules](#basic_rules)
* [Example 1: C-string functions](#example_1)
* [Constant expression](#constant_expression)
* [Example 2: Assertions in compile time evaluations](#example_2)
    * [Compile-time assertion example](#compile_time_assertion_example)
* [Example 3: Compile time objects](#example_3)
* [Example 4: Verifying configuration values at compile time](#example_4)
* [Example 5: Compile time unit testing](#example_5)
* [Constant evaluation drawbacks](#constant_evaluation_drawbacks)

<br></br>
<a name="goal"/>
# Goal
Constant expressions, at its core, allows moving runtime calculations to compile time.

They are helpful for:
1. Improving runtime performance - As the calculation will be made in compile time instead of during runtime.
2. Better error handling - It's much easier to detect errors during compile time than during runtime.


<br></br>
<a name="terminology"/>
# Terminology
The standard is a bit wordy and verbose regarding the rigorous definitions of constexpr, compile-time evaluations and so on.

In here I use the non-official "constexpr context" to mean the execution of a "constant expression" (An expression that can be evaluated at compile time).

It's used interchangeably here with "constant evaluation" and "compile-time evaluation" - They all mean the same thing.

<br></br>
<a name="basic_rules"/>
# Basic rules
Obviously, not every expression is a valid constant expression (e.g. Calling a function defined in another translation unit can never be called during compile time).

The list of things that are allowed (Or more precisely, not allowed) in a constant expression is quite long, and each new release of C++ changes the rules:

For example, in C++20 it was made possible to do the following during a constexpr context:
1. Call a **virtual** method (The compiler basically performs [devirtualization](../Devirtualization/devirtualization.md) for the call site).
2. **Dynamically** allocate memory (Acts as if the allocation succeeded).  
    This allows to create **constexpr** `std::string` and `std::vector`, and use ones in a constexpr context.

But still some things are not allowed: For example calling a non-constexpr function, or throwing exceptions during constant evaluation.

For example, the following:
``` cpp
int func2() {
    return 3;
}

constexpr int func(int a) {
    int b = func2();
    return a + b;
}

constexpr int val = func(3);
```

Will not compile. If instead `func2()` was defined as constexpr then it'll work.

But, it's important to know that only the **flow** of constexpr context must be evaluable at compile time - Any other code (That's not actually evaluated) is allowed to not be evaluable during compile time.

For example, if we change the above `func()` function to:
``` cpp
constexpr int func(int a) {
    if (a > 0) {
        return 1;
    }

    int b = func2();
    return a + b;
}
```

Then while `func2()` is not a constexpr function, calling `func(3)` **can** be evaluable during compile time (Since it will immediately return 1)

Therefore, the line:
``` cpp
constexpr int val = func(3);
```

Will compile just fine [(:star:)](#star_1)

<br></br>
<a name="star_1"/>
(:star:)
> There's a bug in GCC 5 (Up until GCC **9**!) in which such a code will actually **not compile** - The definition of `func()` will cause a compilation error, regardless on whether or not it's called (And with what values).
>
> In order to get around this bug, such `if` statements must be transformed to use the _conditional_ operator ("?" and ":").
>
> For example, `func()` will need to be written as:
> ``` cpp
> constexpr int func(int a) {
>     int res = (a > 0) ? 1 : func2() + a;
>     return res;
> }
> ```
>
> This way calling `func(3)` is a constant expression, while still `func(-1)` is not.
>
> Of course not every if statement can easily be converted to use the conditional operator, but that's what we have right now..

<br></br>
An important thing to remember about constexpr functions is that they can be called from a **non-constexpr context**.

i.e. The following:
``` cpp
constexpr int func(int a) {
    return a + 1;
}

volatile int a = 1;
int b = func(a);  // Compiles fine
```

will compile fine, even though `a` is not a constexpr value, meaning the call to `func(a)` will **not** happen in a constexpr context.

While this has its benefits (Removes the need to have duplicates implementations of the function - One for constexpr context and one for not), it can complicate some other features we would like (More on that later).

<br></br>
<a name="example_1"/>
# Example 1: C-string functions
Let's look at a simple constexpr function that calculates `strlen()` [(:star:)](#star_2):
``` cpp
constexpr size_t constexpr_strlen(const char* str) {
    size_t len = 0;

    while (*(str++)) {
        ++len;
    }

    return len;
}
```

It's important to note a few things about this function:
1. While this function is marked as constexpr, and presumebly will be used in a constant expression (More on that later), it does **NOT** mean that all of the variables defined in it must be **marked as constexpr**.
2. Furthermore, it's allowed to have **non-const** variables - it's possible to **modify** variables defined in a constexpr functions.

Another example is a constexpr function for `strcmp()`:
``` cpp
constexpr int constexpr_strcmp(const char* s1, const char* s2) {                                                                  
    while (*s1 && *s2 && (*s1 == *s2)) {
        ++s1;
        ++s2;
    }

    return (*s1 - *s2);
}
```

<br></br>
<a name="star_2"/>
(:star:)
> GCC and other major compilers have implementation for `strlen()` (And many other C-functions) as a **builtin function** inside the compiler (Their names are prefixed with "__builtin_". e.g. `__builtin_strlen()`).
>
> Furthermore, by default these compilers may translate a call to one of these functions into their builtin counterpart:
> ``` cpp
> size_t s = strlen("123");
> ```
> may (Conceptually) be translated to:
> ``` cpp
> size_t s = __builtin_strlen("123");
> ```
> This allows the compiler to perform optimizations on the function call (e.g. Replace the function call with an inline version).
>
> A nice property of these builtin functions is that most of them can be evaluated during compile time.
>
> And thus, even though `strlen()` is not a constexpr function (As it's a C function, defined in libc.so), it's possible to write:
> ``` cpp
> constexpr size_t s = strlen("123");
> ```
> And it will compile just fine.
>
> In my opinion it's best to not count on that behaviour, and instead explicitly use `constexpr_strlen()`.  
> This is especially true when one can disable the translation to builtin functions by passing the `-fno-builtin` to GCC, which will cause calls to `strlen()` call the matching function from the standard library.

<br></br>
<a name="constant_expression"/>
# Constant expression
It's important to note that calling a function marked as constexpr does **not** guarantee that the call will be evaluated during compile time.

For example, consider the following program:
``` cpp
int main(int argc, char* argv[]) {
    volatile int a = constexpr_strlen(argv[0]); /** Not a constexpr context */
    return constexpr_strlen("123");             /** Not a constexpr context */
}
```
In the generated assembly, the first `constexpr_strlen()` is called with a non constant expression argument, meaning that it will NOT be evaluated at compile time (The volatile is there just to make sure the compiler will not optimize it out completly).

As for the second call, since the return value is not used in a constant expression, meaning it also not evaluated at compile time.

Since both calls to `constexpr_strlen()` are not evaluated during compile time, it's possible that in the generated assembly both function calls will be present (Notice that compiler optimizations could indeed optimize the 2nd call, but it's not mandatory).

If instead, the program was written as:
``` cpp
int main() {
    constexpr size_t size = constexpr_strlen("123");
    return size;
}
```
Then since the argument to `constexpr_strlen()` is a constant expression, and its return value is used in a constant expression (Initialization of a constexpr variable), then the function call is **required** to be evaluated during compile time.

This means the code will be identical to as if it was written as:
``` cpp
int main() {
    constexpr size_t size = 3;
    return size;
}
```

And in this case no sane compiler will not optimize such a code.
Therefore the generated assembly will **not** contain a call to `constexpr_strlen()`, but instead will be simply something like:
``` asm
mov $0x3, %eax
```

<br></br>
<a name="example_2"/>
# Example 2: Assertions in compile time evaluations
It is possible to assert a condition during a compile time evaluation.

First, it's important to know the differences between a **static** assert and **constexpr** assert [(:star:)](#star_3):
1. **static**  assert accepts a constexpr value as the first input, and produces a compilation error if the value of the boolean is _false_.
2. **constexpr** assert, on the other hand, is usually placed inside a constexpr function and has the following behaviour:
    1. If the function is called in a **constexpr context**, and the flow reaches the assertion:
        1. If the value for the assertion is **false**, then a compilation error is returned. Else, nothing happens.
    2. If the function is **not** called from in a constexpr context, the assert will be converted to a runtime assert (More on that later).

Basically, a **static assert** allows to assert the **return value** of a constexpr function (As well as any constexpr boolean value), while a **constexpr assert** allows to assert conditions _during the flow_ of a **constexpr context**.

A static assert can not be used in places of a constexpr assert, since the argument to `static_assert()` must be a **constant expression**, while on the other hand the arguments to a **constexpr assert** may **NOT** be a **constant expression** (e.g. They are regular, non-constexpr variables) - Even though _calling the function_ that calls the constexpr assert **is** in fact a constant expression.

For example:
``` cpp
constexpr uint64_t to_unsigned(int64_t val) {
    constexpr_assert(0 <= val);
    return static_cast<uint64_t>(val);
}
```

Since `val` is not marked as being **constexpr** (It can't, since it's a function parameter), then it can't be the input to a static assert. On the other hand, it's a perfectly legal argument to a **constexpr assert**.

We'll see some more uses for **constexpr assertions** later on.

<br></br>
<a name="star_3"/>
(:star:)
> The term "**constexpr** assert" is not official term. For us it simply means an assertion that can be used inside a constexpr function, and have the aforementioned beahviour.
<br></br>

The language doesn't have builtin support for _constexpr_ assertions (As there are multiple ways to implement one (See below), and none of the options are any better than the other).  
Therefore, we must roll our own implementation (It's not complicated, though).

Before we go and implement constexpr assertions, one must remember that a constexpr function can still be called in a non-constexpr context. In such cases, it's not possible to halt the compilation stage, as it's happening during runtime.

Therefore, one must define how they should handle such errors during runtime. The only options in the language are to either throw an exception, or issue a runtime assertion.

Luckily, throwing an exception and **firing** an `assert()` are both not allowed to happen in a constexpr context :) [(:star:)](#star_4)

This means that if during a constant evaluation an exception is thrown (Or an `assert()` is fired), compilation will halt.

Therefore, we have 2 ways to implement compile-time assertions:
1. Throwing an exception:
    ``` cpp
    #define constexpr_assert(cond) (cond)? (void)0 : throw 1
    ```

2. Calling `assert()`:
    ``` cpp
    #define constexpr_assert(cond) assert(cond)
    ```

Since we don't use exceptions in my workplace, we use the 2nd option.

<br></br>
<a name="star_4"/>
(:star:)
> As said before, calling a non-constexpr function during a constexpr context is not allowed.
>
> Therefore, it might seem odd that simply calling `assert()` as a "constexpr assert" will work, since `assert()` is obviously not a constexpr function, and thus calling is, even if the condition is true, should not be allowed (Like calling any other non-constexpr function).
>
> Luckily the standard comes into our help here: To quote the latest current C++20 draft [9.2.5.6] n4849:
>
>     "An expression assert(E) is a constant subexpression ... if ...
>     — E ... is a constant subexpression that evaluates to ... true"
> This basically means that calling `assert(true)` inside a constexpr context is allowed.
>
> Normally this is not possible for regular functions (As it doesn't matter what the values that are being passed to the non-constexpr function are, all that matters is that it will cause a call to a non-constexpr function, which again is not allowed in a constexpr context).
>
> But `assert()` is special, as it's not really a function - Most often it's implemented as a **macro**. Something like:
> ``` cpp
> #define assert(cond) (cond)? (void)0 : __assert_impl(cond)
> ```
> Which indeed shows that if `cond` is _true_, then what will run is indeed valid in a constexpr context


<br></br>
<a name="compile_time_assertion_example"/>
## Compile-time assertion example
Consider : [(:star:)](#star_5)
``` cpp
enum class COLORS { RED, GREEN, BLUE };

constexpr COLORS to_color(const char* color_str) {
    if (0 == constexpr_strcmp("red", color_str)) {
        return COLORS::RED;
    }
    if (0 == constexpr_strcmp("green", color_str)) {
        return COLORS::GREEN;
    }
    if (0 == constexpr_strcmp("blue", color_str)) {
        return COLORS::BLUE;
    }

    bool dummy = false;
    constexpr_assert(dummy);
}
```

<br></br>
<a name="star_5"/>
(:star:)
> The use of the `_dummy` variable is again to bypass the same previously mentioned GCC -Like the previously mentioned bug, here the call to `assert()` is uncoditional - It is interpreted as if the assert is simply:
> ``` cpp
> assert(...);
> ```
> And since `assert()` is not a constexpr function, GCC complains due to the same bug.
>
> The use of `_dummy` makes the call to `assert()` conditional, conditioned on a condition that is **not a constant expression** (Marking `_dummy` as constexpr will again trigger the same bug)..

<br></br>
Calling this function like so:
``` cpp
constexpr COLORS color = to_color("red");
```
will work just fine.

If instead we have:
``` cpp
constexpr COLORS color = to_color("not a color");
```
Then since we're **forcing** the function call to happen at compile time (Since we store the return value in a _constexpr_ variable), the assertion will be fired during compile-time, and compilation will halt.

If now we have:
``` cpp
/** constexpr */ COLORS color = to_color("not a color");
```
Then the function call will not happen at compile time, meaning running this code will result in a **runtime** assertion being fired.

<br></br>
<a name="example_3"/>
# Example 3: Compile time objects
We can use custom classes during constexpr evaluation (With some restrictions). Let's implement a basic string object that we can manipulate during constexpr evaluations:
``` cpp
class constexpr_string {
private:
    // Can be templated if wanted.
    static constexpr size_t capacity = 100;

    size_t m_len;
    char   m_arr[capacity + 1];

public:
    // Initializes 'm_arr' to zeros = NULL terminators
    constexpr constexpr_string()
     : m_len(0), m_arr()
    { }

    // Initializes 'm_arr' to zeros = NULL terminators
    template<size_t N>
    constexpr constexpr_string(const char (&str)[N])
     : m_len(N-1), m_arr()
    {
        // N-1, since N includes the NULL terminator
        static_assert((0 != N) && (N-1 <= capacity), "String too large");

        for (size_t i = 0; i < N-1; ++i) {
            m_arr[i] = str[i];
        }
    }

    constexpr auto length() const {
        return m_len;
    }

    constexpr void push_back(const char ch) {
        constexpr_assert(m_len < capacity);

        m_arr[m_len++] = ch;
        m_arr[m_len]   = '\0';
    }

    constexpr char& operator[](size_t index) {
        constexpr_assert(index < m_len);

        return m_arr[index];
    }

    constexpr const char* c_str() const {
        return m_arr;
    }
};
```

This class is useful if you want to perform string manipulations during a constexpr context.

Ideally one would use `std::string`, but currently we don't have a compiler that supports constexpr `std::string`. Thus we need to roll our own version  :(

The basic idea behind the implementation is to store the string as a simple buffer (Similar to [SSO](../SSO/sso.md)). But unlike SSO, here the string can not grow larger than the predefined capacity.

Any attempt in a constexpr context to grow the string larger than the predefined capacity will result in a **compilation error**.

We can use this new string like any other object:
``` cpp
constexpr auto get_string() {
    constexpr_string str = "123";
    str[1] = '5';
    return str;
}

constexpr auto str = get_string();
std::cout << str.c_str() << std::endl;
```

An important note about the implementation:

Another thing that is not allowed to happen in a constexpr context is any **Undefined Behaviour** (e.g. Signed integer overflow, nullptr dereference etc..).  
The compiler actually checks for any Undefined Behaviour when it executes in a constexpr context, and halts compilation if one is encountered.

In the `constexpr_string` example, there are compile-time assertions in `push_back()` that we don't overflow the buffer we have.

It's worth saying that accessing an array out of its bounds is in fact **Undefined Behaviour**, which means that technically there's no need for the `constexpr_assert()` there, as calling `push_back()` on a `constexpr_string` that filled its local buffer during a constexpr context will halt compilation.

In my opinion it's best to still have these asserts because:
1. It's explicitly shows that this case is handled, making understanding the code easier.
2. Will make this class useable in non-constexpr evaluation contexts (i.e. It will assert, instead of overflowing the buffer)

<br></br>
<a name="example_4"/>
# Example 4: Verifying configuration values at compile time
One powerful usage of constexpr is to verify **at compile time** the validity of our configuration values.

If some programmer adds a new configuration value, we want to **abort compilation** if that configuration value is not valid (Instead of letting him wait for a runtime error to show up, assuming he'll even test a usage of this new configuration value).

In my workplace we have a "framework" to assert such conditions during compile time. We'll build this "framework" gradually following an example.

To start our example, consider we have an array of strings representing IDs:
``` cpp
constexpr const char* possible_ids[] = { "12a", "4bz", "1a9" };
```
And in our example, each ID **must** be exactly 3 characters long, and each ID **must not** start with a '0'.

We thus will like to verify this at compile time.

One possible way to implement it is to simply `static_assert()` it as follows:
``` cpp
constexpr bool _check_possible_ids_valid() {
    for (const char* id : possible_ids) {
        if (3 != constexpr_strlen(id)) {
            return false;
        }

        if ('0' == id[0]) {
            return false;
        }
    }

    return true;
}

static_assert(_check_possible_ids_valid(), "Configuration is invalid");
```

While this will indeed work, it has 1 major drawback: We don't know _**WHY the check failed**_: We can't distinguish between failing since the configuration value does not contain exactly 3 characters, and if it starts with a '0'.

While with this small example it might not seem important, for a more complicated verification with multiple "sub-assertions" in it, knowing what exactly is wrong with the value can be very helpful in fixing it.

Of course it's still possible to have this behaviour by only using `static_assert()`, for example by splitting the verification function into multiple sub-functions, each one verifying only a single condition and assert each one of them alone.  
But, this will become very cumbersome and verbose.

Instead, we'll go with using `constexpr_assert()`, which will result in a more "natural" way of writing the verification code (i.e. No need to split the verification function), and it will not be much more verbose than the **first** `static_assert()` version, while also allowing us to know exactly **where** the verification failed.

We thus rewrite `_check_possible_ids_valid()` function to use `constexpr_assert()`:
``` cpp
constexpr void _check_possible_ids_valid() {
    for (const char* id : possible_ids) {
        constexpr_assert(3 == constexpr_strlen(id));
        constexpr_assert('0' != id[0]);
    }
}
```
The idea here is that if compilation fails, we can look at the **error trace** to see on which **line** the compiler complains, and thus know which specific assertion was fired [(:star:)](#star_6)
> For example, if we add the new configuration value "1234" we will get the following error:
> ![](imgs/bad_1234.png?raw=true)
> 
> As you can see, we can see from the error trace that it failed since the length of the ID is not 3.
> 
> If we add the  value "012", we'll get:
> ![](imgs/bad_012.png?raw=true)
> 
> Which now tells us it failed since the ID begins with a '0'.
> 
> It's not pretty, but it works..

Sadly, there's no way to add a custom message to a `constexpr_assert()` that will be displayed if compilation fails (Like in a `static_assert()`).  
Therefore, if an extra explanation is needed, we usually add a comment around the assert.

<br></br>
<a name="star_6"/>
(:star:)
> Some adjustments to the code need to be made for the errors to actually occur - Keep reading :)

<br></br>
So we add this function to one of the translation units, compile the code, and... Nothing happens, as expected, since all the IDs are valid.

Now, we _test_ our **code assertion function** (It's always best to test your code, even your testing code :) ), and add "1234" to `possible_ids`:
``` cpp
constexpr const char* possible_ids[] = { "12a", "4bz", "1a9", "1234" };
```
We again compile our code and... Again **nothing happens**!

This makes sense, as no one is actually calling this function.

One might be tempted to fix this issue by explicitly calling this function somewhere in the code (e.g. In one of the main init functions):
``` cpp
int main() {
    // ...
    _check_possible_ids_valid();
    // ...
}
```
Even after adding this call the code will still compile. The only difference is that now if the code will actually run a **runtime assertion** will be fired.

While this is better that nothing happening at all, this is still not what we want, as we want **compilation** to **fail**.

As you might have guessed by reading up until this point, the solution is to call this function in a **constexpr context**.

All we did (Calling this function in one of the init functions) was to call this function during runtime, and as said above the call site will not be called in a constexpr context (Even though the function is marked as constexpr).

<br></br>
We've seen one way to **force** a function call to happen at compile time - Store the return value in a constexpr **variable**:
``` cpp
constexpr auto _x = _check_possible_ids_valid();
```
The variable itself and its value don't matter - What matters is that it's **constexpr**.

Writing this will not compile (Well, not for the right reasons that is), since the function `_check_possible_ids_valid()` returns void.

Therefore, one way to get around this is to make `_check_possible_ids_valid()` return a value:
``` cpp
constexpr int _check_possible_ids_valid() {
    for (const char* id : possible_ids) {
        constexpr_assert(3 == constexpr_strlen(id));
        constexpr_assert('0' != id[0]);
    }

    return 0;
}
```

The return value doesn't really matter, what matters is that we return **something**.

This way everything will work just as we expect - An offending ID will fire the `constexpr_assert()`, and since the function call happens in compile time (Since the return value is stored into `_x`, which is marked as constexpr) that compilation will halt.

<br></br>
And that's basically it, but there's one small "Gotcha" that one needs to know:

Strictly speaking the above has Undefined Behaviour (Well, technically the correct term is that the program is "ill-formed").

Why is that?

To quote the latest current C++20 draft [9.2.5.6] n4849 (Same quote was present in earlier versions):
> For a constexpr function ..., if no argument values exist such that an invocation of the function ... could be an evaluated subexpression of a core constant expression ... the program is ill-formed, no diagnostic required

And gives the following example:
> ``` cpp
> constexpr int f(bool b)
> { return b ? throw 0 : 0; } // OK
> 
> constexpr int f() { return f(true); } // ill-formed, no diagnostic required
> ```
Here the function `f()` is ill-formed, since there are no arguments that can be passed to it, such that it can be evaluated at compile time.

What does it has to do with us?

You'll note that `_check_possible_ids_valid()` does not receive any arguments.

Following the above quote from the standard, since there is only 1 argument values list that can be passed to this function (An empty argument list), it must mean that calling this function **must** be evaluable at compile time, but this is the complete opposite from what we **want** to happen _if the configuration is bad!_

The way we decided to fix this issue is to change `_check_possible_ids_valid()` to accept a dummy bool which by default will be _false_. If the value of `_dummy` is true, then the function will immediately return. Otherwise, it will perform the actual verification:
``` cpp
constexpr int _check_possible_ids_valid(bool _dummy = false) {
    if (_dummy) {
        return 0;
    }

    for (const char* id : possible_ids) {
        constexpr_assert(3 == constexpr_strlen(id));
        constexpr_assert('0' != id[0]);
    }

    return 0;
}
```
With this dummy boolean, it's **technically** possible to **always** have a way to call this function such that it will be evaluable during compile time (Even if the configuration is bad) - By simply passing **true** to `_dummy`.

Thus we are now safe and fully conform to the standard.

Notice that by default `_dummy` is false, which means callers will not be even aware of its existence (And they shouldn't).

Technically speaking we can "hack" this function, and the following:
``` cpp
constexpr auto _x = _check_possible_ids_valid(true);
```
will compile even if the configurations are invalid, but this is not the expected way to call this function, and thus we ignore it.

<br></br>
One final note is about where to define the `_x` variable: It actually does **not** matter where it's defined - Only that it's present **somewhere**. Therefore, we usually put it in a standalone function:
``` cpp
/**
 * Doesn't even have to be constexpr.
 */
__attribute__((used)) // Suppress "function defined but unused" warning.
static void _check_possible_ids_valid() {
    constexpr auto _x = _check_possible_ids_valid_impl();
    (void) _x; // Supress "unused variable" warnings
}
```
And we postfix the name of the actual "verification" constexpr function with "_impl".

<br></br>
Note that we don't even have to call `_check_possible_ids_valid()` from our code!

This is a very useful behaviour of constexpr evaluation - We just need this function to exist and to be compiled, and that is it - We don't need to bloat our init code with all these assertions.

<br></br>
All in all, the complete "verification framework" will look something like:
``` cpp
constexpr int _check_possible_ids_valid_impl(bool _dummy = false) {
    if (_dummy) {
        return 0;
    }

    for (const char* id : possible_ids) {
        constexpr_assert(3 == constexpr_strlen(id));
        constexpr_assert('0' != id[0]);
    }

    return 0;
}

__attribute__((used))
static void _check_possible_ids_valid() {
    constexpr auto _x = _check_possible_ids_valid_impl();
    (void) _x;
}
```

<br></br>
<a name="example_5"/>
# Example 5: Compile time unit testing
Using constexpr evaluation we can perform some unit tests during compile time.

This has the same benefits ofverifying configurations at compile time - The code itself will not **compile** if the tests do not pass.

We use it for example in our `will_add_overflow()` function - It accepts 2 integer types (Of the same type), and returns whether or not adding them up will overflow the type:
``` cpp
template<typename T>
constexpr bool will_add_overflow(const T a, const T b) {
    // Logic not important for now
}
```
Since the logic can be execute at compile time, and indeed the function is marked as constexpr, we can call it in a constexpr context.

We thus would like to "unit test" this function: The test goes over all the possible pairs of uint8_t and int8_t and checks if the function return the expect result. It goes something like:
``` cpp
constexpr bool _check_will_add_overflow() {
    for (uint64_t a = 0; a <= std::numeric_limits<uint8_t>::max(); ++a) {
        for (uint64_t b = 0; b <= std::numeric_limits<uint8_t>::max(); ++b) {
            if (will_add_overflow((uint8_t)a, (uint8_t)b) xor (a + b > std::numeric_limits<uint8_t>::max())) {
                return false;
            }
        }
    }

    // Same for int8_t

    return true;
}

static_assert(_check_will_add_overflow(), "will_add_overflow() failed tests");
```
Notice that in this case we don't really care about **why** the check failed (As this is a rather trivial function we're unit testing), therefore a regular `static_assert()` is fine here, and there's no need for `constexpr_assert()`.

Note that with this approach of doing unit tests at compile time, we don't get the **actual value** that cause the test to fail, but only _whether or not the tests pass_.

To get the values of the parameters that caused the test to fail, one sadly needs to copy the test to another code an run it like any other code.

<br></br>
<a name="constant_evaluation_drawbacks"/>
# Constant evaluation drawbacks
While it's almost always best to move time calculations to be performed during compile time instead of during runtime, it's important to know some limitations on constexpr values and functions:
1. A constexpr variable must be **defined** in the same unit that it's used at. i.e. It's not enough to just **declare** the variable. For example, the following will not compile:
    ``` cpp
    /** file.cpp */
    constexpr int value = get_value();
    
    /** file.hpp */
    extern constexpr int value;
    ```

2. Similarly, for a constexpr function to be useable in a constexpr context, it must be **defined before** it's used. For exammple, the following will not compile:
    ``` cpp
    constexpr int get_value();
    
    constexpr int value = get_value();
    
    constexpr int get_value() {
        return 3;
    }
    ```

Given the above restrictions, in order for a constexpr variable to be used in multiple translation units it must be **defined** in all of them.

Continuing with the above examples, if one wants to use `value` in multiple translation units, it will (Most likely) be defined in a header file:
``` cpp
/** file.hpp */
constexpr int value = get_value();
```
This issue with this is that now **every** translation unit that transitively has `#include file.hpp`, will have the definition of `value` in it.

This means, that the function `get_value()` will be called **for each** translation unit it's defined at (Which can be even the whole project).

This can become problematic if the function `get_value()` is **slow** - Since even if optimizations are turned on, compile time expressions are usually **not optimized at all** (Basically, the compiler runs as if it interprets the constexpr expressions)

Therefore, if the function is rather slow, and due to the usage of header files it gets called from multiple translation units, the **total compilation time of the whole program will increase**, in respect to the number of translation units this expression is defined at.

Multiplying this by a lot of constexpr values that can be defined in such a way, and soon enough compilation can be unbearably slow.

The same is true for the _assertions, configurations verifications and unit tests_ mentioned in previous examples.

Therefore, the best thing to do in order to remedy this is to **not** put these definitions in a header file (If possible, of course).

For example, configurations verifications and the unit tests can actually be defined in a new, dedicated translation unit (Because remember that it's enough to have the checks be defined **somewhere** in the code, it doesn't matter **where** - As long as the file gets compiled of course).
