# Portable C/C++ Calculations?
Easy peasey! Just stop using `int`, and use fixed width integers like `int16_t`. Right?

I wish that was the case. Let's look at a simple example.

> If you are an experienced C programmer that understands all the C/C++ foot-guns, you might want to skip to the bottom section for some new ideas.

Here's a small function that multiplies two numbers, and divides by 1024. To make the code more portable, we'll use `uint16_t` instead of `int`.

```c
uint16_t calc_1(uint16_t a, uint16_t b)
{
  return a * b / 1024;
}
```

We'll run some simple test code on an Atmel AVR (Arduino Uno) and also an STM32.

```cpp
// test code:
uint16_t a = 1000;
uint16_t b = 2000;
uint16_t result = calc_1(a, b);
Serial.print("calc_1 result: ");
Serial.println(result);
```

If you do this math with a calculator, you'll get `1953.125` so we should be expecting `1953` due to integer truncation.

On the STM32, the result is `1953` as expected.

Maybe we even write some unit tests that run on our 64 bit computer and they agree with the STM32. We're good to go, right?

# Enter the AVR

On the AVR, the result is `33`!

What the flip!? What's going on? We used fixed width integers!

Well... the non portable `int` behavior snuck in without us noticing. In C, before arithmetic operations are performed, operands **smaller** than `int` are promoted to `int`.

When this promotion to int happens, this is basically what our program does:

```c
// 32 bit int STM32 scenario
uint16_t calc_1(uint16_t a, uint16_t b)
{
  int promoted_a = a; // 32 bit int
  int promoted_b = b; // 32 bit int
  int interim_result = promoted_a * promoted_b;  // 32 bit int
  // interim_result = 0b0001_1110_1000_0100_1000_0000 (24 bits required)
  
  // divide by 1024 is like right shift 10 times
  interim_result = interim_result >> 10;
  // interim_result = 0b0111_1010_0001

  uint16_t result = (uint16_t)interim_result; // implicit truncate to 16 bits
  return result;
}
```

When the operand is the same size or larger than `int`, there is no promotion. This is basically what our program does:
```c
// AVR scenario (16 bit int)
uint16_t calc_1(uint16_t a, uint16_t b)
{
  uint16_t interim_result = a * b; // 16 bit int. OVERFLOW!!!
  // can only hold lower 16 bits of ideal 24 bit result.
  // interim_result = 0b1000_0100_1000_0000 (33,920 dec)
  
  // divide by 1024 is like right shift 10 times
  interim_result = interim_result >> 10;
  // interim_result = 0b10_0001 (33 dec)

  return interim_result;
}
```

Well fork me! We're not getting portable math just by using fixed width integers.

<br>
<br>

# So how do we get portable math in C?

## Cast your way to portable wrongness
Many new C programmers cast absolutely everywhere even when not needed because they don't yet understand some of the tricky parts of C (promotions, conversions...). Casts seem to fix weird issues, so more is better. This is a bad habit because it makes the code harder to read and understand.

```c
// lots of casts, wrong answer
uint16_t calc_2(uint16_t a, uint16_t b)
{
  return ((uint16_t)((uint16_t)a * (uint16_t)b)) / 1024;
}
```

We still get the wrong result of `33`, but at least it is the same wrong on all platforms :)

We can achieve the same portable wrong result with one cast in this _particular_ case.
```c
// fewer casts, still wrong
uint16_t calc_3(uint16_t a, uint16_t b)
{
  return (uint16_t)(a * b) / 1024;
}
```

Note that we don't need to wrap `(uint16_t)(a * b)` in parentheses like `((uint16_t)(a * b))` because the cast has higher precedence than the division. Adding extra parentheses is not the end of the world, but is also extra just noise.


## Widening Cast
We essentially need to widen (increase bits of) the operands to a type that is large enough to hold the interim result. In this case, we need to widen to `uint32_t` because the interim result may be up to 32 bits.

```c
// prints the correct result!
uint16_t calc_4(uint16_t a, uint16_t b)
{
  return (uint32_t)a * b / 1024;
}
```

### Why not cast `b` as well? [32 bit int]
If on a 32 bit int machine...

* Casting b is unnecessary since `b` gets converted to `uint32_t` to match the **rank** of `(uint32_t)a` before the multiplication.
* The cast `(uint32_t)a` has higher precedence than the multiplication, so the cast is applied to `a` before the multiplication.
* The interim result of type `uint32_t` is then carried through the rest of the calculation.

```c
// when `int` is 32 bits (or less), this is what the compiler does.
uint16_t calc_4(uint16_t a, uint16_t b)
{
  // `(uint32_t)a * b`
  uint32_t cast_a = a;     // 32 bit
  uint32_t promoted_b = b; // 32 bit
  uint32_t interim_result = cast_a * promoted_b;  // 32 bit int
  // interim_result = 0b0001_1110_1000_0100_1000_0000 (24 bits required)
  
  // divide by 1024 is like right shift 10 times
  interim_result = interim_result >> 10;
  // interim_result = 0b0111_1010_0001

  uint16_t result = (uint16_t)interim_result; // implicit truncate to 16 bits
  return result;
}
```

### On a 64 bit int machine
Casting `b` is still unnecessary since `b` gets promoted to int (`int64_t`) before the multiplication.

In fact, `a` will also end up getting promoted to `int` (`int64_t`) before the multiplication with `b`.

```c
// when `int` is 64 bits, this is what the compiler does.
uint16_t calc_4(uint16_t a, uint16_t b)
{
  // `(uint32_t)a * b`
  uint32_t cast_a = a;         // 32 bit
  int promoted_cast_a = cast_a // 64 bit int
  int promoted_b = b;          // 64 bit int
  int interim_result = promoted_cast_a * promoted_b;  // 64 bit int
  // interim_result = 0b0001_1110_1000_0100_1000_0000 (24 bits required)
  
  // divide by 1024 is like right shift 10 times
  interim_result = interim_result >> 10;
  // interim_result = 0b0111_1010_0001

  uint16_t result = (uint16_t)interim_result; // implicit truncate to 16 bits
  return result;
}
```

## Casts can hide mistakes
C casts are very powerful, but need to be treated with care as can easily hide mistakes. This usually isn't a big deal if you are writing unit tests though.

If for some reason we refactored our function to instead take a `pointer` to a `uint16_t`, we might make a mistake like below. Many compilers (even expensive ones like IAR) will not warn about this because we've told the compiler that we know what we're doing with the cast.


```c
// Before refactor to take a pointer
uint16_t calc_4(uint16_t a, uint16_t b)
{
  // cast to widen `a` to 32 bits
  return (uint32_t)a * b / 1024;
}
```

```c
// Oops!
// After refactor to take a pointer.
uint16_t calc_5(uint16_t *a, uint16_t b)
{
  // original cast to promote `a` is still there,
  // but now it is hiding a mistake.
  return (uint32_t)a * b / 1024;
}
```

A safer way to do widen an integer is with a tiny helper function.

```c
// this is optimized away when any optimization is enabled
static inline uint32_t widen_to_u32(uint32_t a)
{
    return a;
}

uint16_t calc_6(uint16_t a, uint16_t b)
{
  return widen_to_u32(a) * b / 1024;
}
```

The widening function will prevent us from accidentally treating a pointer as an int, but can actually detect a lot of misuse (like receiving incompatible integer types) if we enable `-Wconversion` (GCC).

## `-Wconversion` helps
Note! `-Wconversion` is not enabled in `-Wall` or `-Wextra` on GCC. You need to add it explicitly.

With `-Wconversion`, we need to be explicit with our truncation for the result.

```c
uint16_t calc_7(uint16_t a, uint16_t b)
{
  return (uint16_t)(widen_to_u32(a) * b / 1024);
}
```

```c
// or if you are cast shy:
static inline uint16_t trunc_u_to_u32(uint64_t a)
{
    return (uint16_t)a;
}

uint16_t calc_7b(uint16_t a, uint16_t b)
{
  return trunc_u_to_u32(widen_to_u32(a) * b / 1024);
}
```



<br>
<br>
<br>



# C/C++ mixing signed and (gasp!) unsigned
Here's some more non-portable code that you might see in the wild.

```c
// implementation defined behavior in C
uint16_t a = 1000;
int16_t b = -1;
bool a_is_greater = a > b; // ???
```

What is the result of `a_is_greater`? It depends on the platform.

On AVR, `a_is_greater` is false. On platforms with 32 bit (or larger) `int`, `a_is_greater` is true.

Why? Well because C/C++ are really old languages and this one particular unfortunate decision made sense back in the day, but really doesn't anymore.

The issue is that when comparing signed and unsigned integers of `size(int)`, the signed integer is converted to unsigned. Yep, a negative signed number can end up looking like a huge unsigned number.

This page has lots of good info on C promotions and conversions: https://www.geeksforgeeks.org/type-conversion-c/

## C/C++ Solution - Widen Operands
Luckily, compilers are pretty good at warning about this if you enable it (not a default). You can enable it with `-Wsign-compare` on GCC.

You need to manually widen the signed and unsigned arguments.

```c
uint16_t a = 1000;
int16_t b = -1;
bool a_is_greater = (int32_t)a > (int32_t)b; // true
```
Or you can just widen one side and let promotion handle the other. All the typical issues with casting can still occur. You could create a widening function like shown earlier to be safer.


## C/C++ Solution - Test Signed
If we don't want to widen the operands to something that can hold both signed and unsigned values, we can test the signed integer for negative first and then compare both as unsigned.

```c
uint16_t a = 1000;
int16_t b = -1;
bool a_is_greater = b < 0 || a > (uint16_t)b; // true
```

Again, watch out for future refactoring or copy paste mistakes where the size of `b` is changed, but the unsigned cast is not.

<br>
<br>
<br>

# Simpler With Fin
Find this confusing? Many people do. And even if you do understand it, it still takes time to explain to others and catch in code reviews...

Fin code is much easier to read and understand:
* u16 types use u16 math...
* fin math is portable across platforms by default.
* if you need more width, be explicit.
* there are no implicit promotions to `int`.
* there are no implicit truncations.
* is generally safer

## fin - safe mixing signed and unsigned
In `fin`, this is a non-issue. Nice and simple.

```c
// fin - fully defined and expected behavior
u16 a = 1000;
i16 b = -1;
bool a_is_greater = a > b; // true!!!
```

## fin - no implicit promotion to `int`
Promoting to system defined `int` results in non portable code, is confusing at times and can lead to unexpected results. We want our `fin` code to work correctly on any embedded platform regardless of `int` size.

If you want something to be widened to a larger size in fin, you simply need to ask for it.

```cs
// fin - potential syntax:
u16 a = ...;
u32 temp = (u32)a;   // fin casts are much safer than C casts
u32 temp = a.u32();
u32 temp = a.to_u32();
u32 temp = a.widen_u32();
u32 temp = a.up_u32();
```

## fin - no implicit truncation
C (without extra, extra, extra warnings) will happily truncate your data without telling you. Often this is what we want, but sometimes this does bite us. It can lead to unexpected results, undefined behavior, and bugs.

In fin, you need to be explicit about truncation (narrowing conversions).

```cs
// fin - potential syntax:
u32 temp = ...;
u16 result = (u16)temp;  // fin casts are safer than C casts
u16 result = temp.wrap_u16(); // truncates top bits
```

## fin - trapping overflow/underflow
In C, signed overflow/underflow is undefined behavior. This means that the compiler can do whatever it wants. It can even make demons fly out of your nose. This is not a joke. It is a real thing. [See here](https://blog.regehr.org/archives/213).

In fin, overflow/underflow will be caught by default. This is a good thing. It is much better to know that you've done something wrong than to have the compiler do whatever it wants.

How we will achieve this is still for debate.

## fin - panic or exception?
zig and rust both panic (AKA kill the program) when they detect an integer overflow/underflow.

This makes sense for the needs of those languages, but I think it is a terrible design choice for embedded. Embedded systems are often safety critical and need to keep running even if a bug occurs.

Killing the program should NOT be the default behavior.

If my code is controlling a tractor and a bug in a data logging system allows a variable to overflow, I want the program to have the choice on how to react:
* I might want to shutdown the tractor in a controlled fashion (not just reboot and slam to a halt).
* I might want to send a warning to the user, disable the data logging system, and keep running as the data logging system is not crucial to operation.
* I might want to restart the data logging system and keep running.
* ...

## more to come...
