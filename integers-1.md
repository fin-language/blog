# Intro
While I love C, it does have a number of really annoying areas. One of them is integer math.

Fin takes an approach to math similar to Rust, but much less verbose and yet safer.

The main goal is to make it easy to read/write portable and safe fin/C code.

## No Implicit Truncation
If you want to truncate a variable, you have to explicitly do so in fin.

In C, it is perfectly legal to write buggy code like below:

```c
// C99
// silent truncation from u32 to u8
uint8_t add(uint32_t a, uint32_t b)
{
  uint32_t result = a * b;
  return result; // !!! silent truncation from u32 to u8
}
```

```c
// C99
// silent conversion from i32 to u8
void use_u8(uint8_t a)
{
  //...
}

int32_t a = -55555555; // some value that doesn't fit in u8
use_u8(a); // !!! silent conversion from i32 to u8
```

You won't get GCC `C99` warnings about the above mistakes with `-Wall`, `-Wextra`, or `-Wpedantic`. You have to explicitly enable `-Wconversion` to get a warning.

But as soon as you do that, you get a ton of warnings about perfectly legal code like:

```c
// C99
uint8_t b = 0;
b = b + 10; // GCC warning: conversion to ‘uint8_t from ‘int’ may alter its value
```

And now we have to put casts absolutely everywhere which I have trouble stomaching... 🤢
```c
// C99
uint8_t b = 0;
a = (uint8_t)(b + 10); // to avoid annoying warning with -Wconversion
```

Once you start to feel the pain of `-Wconversion`, you can easily undertand why it was left out of `-Wall` and `-Wextra`. It's just too noisy.

With fin, we get safety without excessive warnings and noise.


## More Expressive Operations
C casts are a powerful, yet blunt tool. They can easily hide mistakes because the same syntax is used for many different operations:

| Conversion Type                             | C Code                |
| ------------------------------------------- | --------------------- |
| data type widening                          | `(uint32_t)my_u8`     |
| data type narrowing (no expected data loss) | `(uint8_t)my_u32`     |
| data type narrowing and truncation          | `(uint8_t)my_u32`     |
| data type conversion                        | `(uint8_t)my_i8`      |
| imprecise conversion                        | `(float)my_u32`       |
| data re-interpretation                      | `(uint32_t)my_u8_ptr` |
| remove const                                | `(uint8_t*)my_ptr`    |
| remove volatile                             | `(uint8_t*)my_ptr`    |
| etc.                                        | ...                   |

When looking at a c-style cast `(uint16_t)my_var`, it is not always apparent at a glance which of the above operations is being invoked. What they all have in common though is telling the compiler:
> _"Trust me compiler! I know what I'm doing. Don't check for errors."_

That may be true at the time of writing the code, but there's nothing in the language to ensure that it stays that way. New programmers inheriting the code, time pressures, refactoring, new features... can easily lead to mistakes where the compiler can't help us because we told it not to.

Most languages ([even C++](https://stackoverflow.com/questions/1609163/what-is-the-difference-between-static-cast-and-c-style-casting)) have moved away from c-style casts because they are too broad and "dangerous". Sometimes maintenance programming results in errors where a cast isn't updated correctly. What used to be a widening cast, becomes a narrowing/truncating cast, and now we have a bug.

With `fin`, we have distinct operators that allow us to express our intent. This makes it easier to read and write code that is less susceptible to many issues.


## Safe Mixing Of Signed & Unsigned
Lot's of issues with C here. Common source of bugs.

```c
// C - implementation defined behavior
uint16_t a = 1000;
int16_t b = -1;
bool a_is_greater = a > b; // ??? need to know int size
```

Fin solves this completely. More below.

## No Implicit Promotion To `int`
C implicitly promotes smaller integer types to `int` when performing arithmetic operations or comparisons. This makes it difficult to write portable/consistent code as you have to always be thinking about the potential size of `int` (16/32/64 bit). See STM32/AVR example below.

In `fin`, you can easily and safely widen to a larger type.


<br>
<br>


# Tiny Example
Here's a small function that multiplies two numbers, and divides by 1024. This code may not behave consistently cross platform. It sure looks like it should, but it doesn't because of the implicit promotion to `int`.

```c
// c99 - non-portable
uint16_t calc_1(uint16_t a, uint16_t b)
{
  return a * b / 1024;
}
```

This is the basic test code we will run on each platform:

```cpp
// c99 test code
uint16_t a = 1000;
uint16_t b = 2000;
uint16_t result = calc_1(a, b);
```

If you do this math with a calculator, you'll get `1953.125` so we should be expecting `1953` due to integer division.

On an STM32, the result is `1953` as expected. On the AVR, the result is `33` because it has a 16 bit int which overflows.

To fix this, we explicitly convert `a` (and/or `b`) to u32 before doing the math. We should also cast the result back down to u16 to make it clear that we are truncating the result and avoid compiler warnings.

```c
// c99 - fixed
uint16_t calc_7(uint16_t a, uint16_t b)
{
  return (uint16_t)((uint32_t)a * b / 1024);
  // what is the intent of the `(uint16_t)` cast though?
}
```

But... is the `(uint16_t)` cast to avoid a compiler warning because truncation should never happen?

Or is the truncation/wrapping behavior desired? It needs a comment to explain.

## `Fin` Solution
The below `fin` code is equivalent to the fixed C99 code above. Later sections explain this code, but just take a quick peek for now. It's pretty easy to understand what is going on.

```cs
// fin
u16 calc_7(u16 a, u16 b)
{
  return (a.u32 * b / 1024).narrow_to_u16();
  // - `a.u32` is a safe widening of a to u32
  // - `b` is implicitly widened to u32 to match `a.u32`
  // - `.narrow_to_u16()` is a narrowing conversion that checks for data loss/truncation.
  //   If truncation happens during C# simulation, an exception is thrown. The generated C code
  //   can either check for truncation or not depending on the math mode. More below.
}
```

<br>
<br>

# `Fin` Specification

Overview of fin conversions:

| Conversion Type                                  | Example             | Fin Code                     | Error              | Notes  |
| ------------------------------------------------ | ------------------- | ---------------------------- | ------------------ | ------ |
| safe data type widening                          | `u8` to `u32`       | `my_u8.u32`                  | No error possible  |        |
| " "                                              | `u32` from `u8`     | `u32.from(my_u8)`            | " "                |        |
| " "                                              | `u8` to `i16`       | `my_u8.i16`                  | " "                |        |
| data type narrowing <br> (no expected data loss) | `u32` to `u8`       | `my_u32.narrow_to_u8()`      | Error if data loss |        |
| "                 "                              | `i8` to `u8`        | `(u8)my_i8`                  | " "                | `CAST` |
| "                 "                              | `i8` to `u8`        | `my_i8.narrow_to_u8()`       | " "                |        |
| "                 "                              | `u8` from `i8`      | `u8.narrow_from(my_i8)`      | " "                |        |
| data type narrowing <br> and wrapping/truncation | `u32` to `u8`       | `my_u32.wrap_to_u8()`        | No error possible  | `NT`   |
| data type re-interpretation                      | `i8` as `u8`        | `my_i8.get_bits()`           |                    |        |
| " "                                              | `u8` as `i8`        | `my_u8.bits_to_i8()`         |                    | `RI`   |
| " "                                              | `u8` bits from `i8` | `my_u8.set_bits_from(my_i8)` |                    | `RI`   |
| imprecise conversions                            |                     | TBD                          |                    |        |
| const/volatile conversions                       |                     | TBD                          |                    |        |

* Note `CAST` - Casts between types in fin are treated the same as a narrowing conversion. It is an error to cast `-1` to `u8` in fin. See bits methods for data reinterpretation if that's what you need.
* Note `NT` - converting from unsigned to signed is implementation defined behavior in C. Will not be initially supported in fin. Users can manually implement it though.
* Note `RI` - implementation defined when converting between signed/unsigned or we can use `memcpy()` (slower, but safer). Probably want a compiler switch here.

## Safe Widening Operators
It is safe to widen to larger types so fin provides succinct properties (`u16 a` example):
* `a.u32` - safe widening to `u32`. Generates to `(uint32_t)a` in C.
* `a.u64` - safe widening to `u64`. Generates to `(uint64_t)a` in C.
* `a.i32` - safe widening to `i32`. Generates to `(int32_t)a` in C.
* `a.i64` - safe widening to `i64`. Generates to `(int64_t)a` in C.
* `a.f32` - safe widening to `f32`. Generates to `(float)a` in C.
* `a.f64` - safe widening to `f64`. Generates to `(double)a` in C.
* ...

Note that `u16` types do <u>NOT</u> have `a.u8`, `a.u16`, `a.i8`, `a.i16` because those are not safe widening operations.

## Explicit Saturation (not yet implemented)
Values can be converted to smaller types in a saturating manner with a number of methods (`u16 a` example):
* `a.sat_u8()`  - converts to `u8` clamping to limits. Function call in C.
* `a.sat_i8()`  - converts to `i8` clamping to limits. Function call in C.
* `a.sat_i16()` - converts to `i16` clamping to limits. Function call in C.

## Explicit Wrapping/Truncation
Unsigned values can be explicitly wrapped/truncated to smaller types with a number of methods  (`u16 a` example):
* `a.wrap_u8()` - wraps/truncates to `u8`. Generates to `(uint8_t)a` in C.

> Note: we will not have wrap functions for signed integers because that is undefined or implementation defined behavior in C. We usually want modulo/wrapping behavior for unsigned values. I can't think of when I would want that with signed values at the time of writing this. If there are good use cases, we can add them.

> Future functions: `u8.wrap(my_u16)` - class methods.

## Narrowing
If math mode (covered in a below section) is unsafe, narrowing conversions will check for overflow in fin/C# simulation/tests, but the generated C code does not. This is useful if size or performance is a concern.

If math mode is user error capturing, narrowing conversions will check for overflow in fin/C# simulation/tests and in the generated C code. This is useful when correctness is a primary concern.

<!-- 
## Explicitly Checked Narrowing
These methods narrow to smaller data types, but they also check for overflow. If the value is too large, they set the `err` parameter to `true`. These methods generate to C function calls. The `err` parameter is only set on overflow (it is not cleared on success).

```cs
// fin
Err err = // stack allocated error object
u16 a = 256 + 1;
u8 b = a.narrow_to_u8(err);
// b == 1, err.is_active == true

err.clear();
b = (a.u32 * b / 1024).narrow_to_u16(err);
```

Why take an `Err` object as a parameter instead of returning a bool? Lots of reasons (allows chaining and combining, easy to read, ...). That said, we will likely eventually add another style for operations that return a `bool` instead of using an `Err` object:
```cs
// fin
u16 a = 256 + 1;
u8 b;
bool success = a.narrow_to_u8(out b);
// b == 1, success == false

success = (a.u32 * b / 1024).narrow_to_u16(out b); // I don't like this one much
// maybe we do it the opposite way like C23 (bool true means overflow).
```
 -->


## Safe Implicit Widening
In the below code, we don't need to do `b.u32` because it is automatically widened to match `a.u32` before the multiplication.

```cs
// fin
u32 calc_7(u16 a, u16 b)
{
  return (a.u32 * b / 1024);
}
```
Rust takes a different approach and requires explicit widening for all operands:

```rust
// rust
fn calc_7(a: u16, b: u16) -> u32
{
  return (u32::from(a) * u32::from(b) / 1024);
}
```

We can add a `fin` option to require explicit widening if desired.


<!-- 
## Implementation Defined Behavior
Converting to unsigned types is always well defined. Converting to signed types is implementation defined behavior in C.

> https://en.cppreference.com/w/c/language/conversion<br>
**Integer conversions**<br>
A value of any integer type can be implicitly converted to any other integer type. Except where covered by promotions and boolean conversions above, the rules are:<br>
\- if the target type can represent the value, the value is unchanged<br>
\- otherwise, if the target type is unsigned, the value 2^b, where b is the number of value bits in the target type, is repeatedly subtracted or added to the source value until the result fits in the target type. In other words, unsigned integers implement modulo arithmetic.<br>
\- otherwise, if the target type is signed, the behavior is implementation-defined (which may include raising a signal)

 -->









## Mixing Signed & Unsigned
Fin takes a sane approach to mixing signed and unsigned types. It is safe to mix signed and unsigned types in arithmetic operations. The result is always a type that is large enough to hold both operands.

```c
// C - implementation defined behavior
uint16_t a = 1000;
int16_t b = -1;
bool a_is_greater = a > b; // ??? need to know int size
```

<!-- TABLE  -->
  <table>
  <tr><td>Fin</td><td>C</td></tr>
  <tr><td>

  ```cs
  // fin
  u16 a = 1000;
  i16 b = -1;
  bool a_is_greater = a > b; // true!!!
  ```

  </td><td>

  ```c
  // Generated C99
  uint16_t a = 1000;
  int16_t b = -1;
  bool a_is_greater = (int32_t)a > (int32_t)b;
  ```
</td></tr>
</table>
<!-- /TABLE  -->


Math conversions:
```c
u8 + i8  -> i16   // i16 can hold all of u8 and i8
u8 + i16 -> i16   // i16 can hold all of u8
... 
u16 + i8  -> i32  // i32 can hold all of u16 and i8
u16 + i16 -> i32  // i32 can hold all of u16 and i16
u16 + i32 -> i32  // i32 can hold all of u16
...
```

<br>
<br>

# Math Mode & Error Handling
Fin currently has 2 math modes:

| Math Mode         | C# simulation error behavior               | C code error behavior |
| ----------------- | ------------------------------------------ | --------------------- |
| unsafe            | throws exception                           | unchecked             |
| user provided err | sets error flag (exception if not checked) | sets error flag       |


## You must choose a math mode
Eventually, fin will default to a safe math mode (that doesn't yet exist), but for now you must explicitly choose a math mode. We haven't decided on the future default safe math mode yet. Will it use lightweight exceptions? Lots to discuss around exceptions, but we don't need to worry about that for now. We have good options already.

If you forget to choose a math mode, fin will throw an exception in C# simulation.

```cs
// fin
public void simple_1()
{
    u8 a = 200;
    a += 1;  // C# exception thrown here!
}
```
```
System.InvalidOperationException : Math mode must be specified for now (until fin default established).
```


## Unsafe Mode
Unsafe will be supported in code generation first.

Example of unsafe mode:

```cs
// fin
public void simple_1()
{
    math.unsafe_mode(); // should be at top of function scope
    u8 a = 200;
    a += 200;  // C# exception thrown here!
}
```

```
System.OverflowException : Overflow! `200 (u8) + 200 (u8)` result `400` is beyond u8 type MAX limit of `255`. Explicitly widen before `+` operation.
```
Because we are using C# for fin simulation, we also get full stack trace information which is really nice.

Generated C code:
```c
// C99
void simple_1(void)
{
    // fin: math.unsafe_mode();
    u8 a = 200;
    a += 200; // no error checking in C code (unsafe mode)
}
```

## User Provided Error Mode
Available for simulation now. Will be supported in C code generation after unsafe mode.

```cs
// fin
public void simple_2()
{
    Err err = mem.stack(new Err()); // stack allocated error object
    math.capture_errors(err);

    u8 a = 255;
    a += 2;  // No exception thrown here! It did however set the err flag.
             // Note: the value of `a` is now `1` because of wrapping.

    if (err.has_error())
    {
        // do stuff
        err.clear();
    }
}
```

You can do multiple operations before checking for errors. The error flag is only set if an error occurs.

```cs
// fin
public void simple_3()
{
    Err err = mem.stack(new Err());
    math.capture_errors(err);

    u8 a = 255;
    a += 2;
    u8 b = a + 6;
    u8 c = a * 2;
    // a == 1, b == 7, c == 2

    if (err.has_error())
    {
        // do stuff
        err.clear();
    }
}
```

You can also pass the `Err` object to helper methods like shown below:

```cs
// fin
private static u8 calc(Err err, u8 a, u8 b)
{
    math.capture_errors(err);
    u8 c = a + b;
    return c;
    // no need to check or clear err here, because the err object was provided to this function.
}

public void simple_4()
{
    Err err = mem.stack(new Err());
    math.capture_errors(err);

    u8 a = 100;
    u8 b = 200;
    u8 c = calc(err, a, b);
    // a == 100, b == 200, c == 44

    if (err.has_error())
    {
        // do stuff
        err.clear();
    }
}
```

### Avoiding Error Flag Pitfalls
It is a common problem in C for developers to forget to check for errors. Fin partially solves this now by throwing an exception in C# simulation if you forget to check for errors.

```cs
// fin
public void simple_2()
{
    Err err = mem.stack(new Err());
    math.capture_errors(err);

    u8 a = 255;
    a += 2;
    // forgot to check for errors here!
}
```
```
finlang.err.ErrMisuseException : Err error must be read and cleared before going out of scope (stack object destructed).
```

You also can't clear an error that hasn't been read.

```cs
// fin
public void simple_2()
{
    Err err = mem.stack(new Err());
    math.capture_errors(err);

    u8 a = 255;
    a += 2;
    err.clear(); // Exception! Can't clear an error that hasn't been read.
}
```
```
ErrMisuseException : Err error not read before cleared. If you really don't care use `disregard_any_error()`.
```

If you really don't care about the error, you can disregard it:
```cs
// fin
public void simple_2()
{
    Err err = mem.stack(new Err());
    math.capture_errors(err);

    u8 a = 255;
    a += 2;
    err.disregard_any_error(); // no exception
}
```

<!-- 

## Overflow, Divide By Zero Detection
You can chain math calls together. If any overflow or divide by zero occurs, the `Err` object will be set.

```c
// fin
u8 calc_stuff(u8 a, u8 b, u8 c, Err err)
{
  return a.add(b, err).div(c, err);
}
```

## Best Of Both Worlds
In the nearish future, we will also be able to write something like the below code which is equivalent to `a.add(b, err).div(c, err)` code. This allows even more natural mathematical expressions, but also captures errors.

```cs
// fin
u8 calc_stuff(u8 a, u8 b, u8 c, Err err)
{
  Math.capture_errors(err);
  return (a + b) / c;
}
```

<br>
<br>

# Near Future Error Handling
In the future, math will be safe by default. If you want to ignore potential errors for speed or code size, you will need to explicitly do so.

## Explicit Ignoring
```cs
// fin
u8 calc_stuff(u16 a, u16 b, u16 c)
{
  Math.unsafe_mode(); // ignore `a+b` overflow, divide by zero
  return ((a + b) / c).narrow_to_u8();
}
```

## Super Explicit Handling
Not recommended or supported initially.
```cs
// fin
u8 calc_stuff(u16 a, u16 b, u16 c, Err err)
{
  return a.add(b, err).div(c, err).to_u8();
}
```


## Explicit Handling (Natural Syntax)
```cs
// fin
u8 calc_stuff(u16 a, u16 b, u16 c, Err err)
{
  Math.capture_errors(err); // capture `a+b` overflow, divide by zero, to_u8() error
  return ((a + b) / c).to_u8();
}
```

<br>
<br>
<br>
<br>


# Math Code Generation Options...
Given the below fin code, what should the generated C code look like?

```cs
// fin
u16 calc(u16 a, u16 b)
{
  return (a.u32 * b / 1024).wrap_u16();
}
```

## C99 - casts
Casts everywhere to ensure portability regardless of int size.

```c
// Generated C99 - casts
uint16_t calc(uint16_t a, uint16_t b)
{
  // fin: (a.u32 * b / 1024).wrap_u16();
  return (uint16_t)((uint32_t)((uint32_t)((uint32_t)a * (uint32_t)b) / 1024) );
  // Note: we could eventually be smarter and remove some casts above.
}
```

Pros:
- No function call overhead

Cons:
- Lots of casts, hard to read.

## C99 - inline functions

```c
// Generated C99 - inline functions
uint16_t calc(uint16_t a, uint16_t b)
{
  // fin: (a.u32 * b / 1024).wrap_u16();
  return wrap_u32_to_u16( div_u32( mult_u32(a, b), 1024 ) );
}

// in another header file...
static inline uint32_t mult_u32(uint32_t a, uint32_t b) { /*...*/ }
static inline uint32_t div_u32(uint32_t a, uint32_t b) { /*...*/ }
static inline uint16_t wrap_u32_to_u16(uint32_t a) { /*...*/ }
```

```c
// Generated C99 - inline functions

//fin: a.add(b, err).div(c, err);
u8_checked_div( u8_checked_add(a, b, err), c, err);
```

Pros:
- easy to read

Cons:
- function cost in debug builds (although none when optimizations are enabled)
- will need some way of ensuring function names don't clash with user code



## C99 - hand inlined functions
Pros:
- very easy to debug and follow
- great for code coverage (if all branches are reachable)

Cons:
- harder to implement
- large code size

Pseudo code:
```c
//fin: a.add(b, err).div(c, err);
uint8_t temp = a + b; // allow roll over
if (temp < a)
{
	if (err.code == NO_ERR)
		// overflow
		err.code = OVERFLOW;
}
else
{
	if (c == 0 && err.code == NO_ERR)
	{
		err.code = DIV_BY_ZERO;
	}
	else
	{
		temp /= c;
	}
}
```


## Decision
Start with whichever is easier to implement. Not bad either way with `fin: ` comment above.
 -->


<!-- 

<br>
<br>
<br>

# Idea Dump

White spacing options. Not sure I like them.

```c
// fin: (a.u32 * b / 1024).wrap_u16();
return (uint16_t)((uint32_t)((uint32_t)((uint32_t)a * (uint32_t)b) / 1024) );

// potential whitespace option:
return (uint16_t)(
          (uint32_t)(
            (uint32_t)(
              (uint32_t)a * (uint32_t)b
            )
            / 1024
          )
        );
```

```c
// fin: (a.u32 * b / 1024).wrap_u16();
return _wrap_u32_to_u16( _div_u32( _mult_u32(a, b), 1024 ) );

// potential whitespace option:
return _wrap_u32_to_u16( 
          _div_u32( 
            _mult_u32(a, b),
            1024 ) );
```

 -->



