# Portable C/C++ Calculations? 🙇‍♂️
Easy peasey! Just stop using `int`, and use fixed width integers like `int16_t`. Right?

Wouldn't that would be nice? Instead, we are going to run some simple code on an STM32 and Atmel AVR (Arduino Uno) to explore some of the simpler foot-guns available to us in C/C++.

![image](https://github.com/fin-language/blog/assets/274012/b9142f74-9a5f-4d4c-bc1e-7c8ddc430d15)

> **Note:** if you are an experienced C programmer that understands all the C/C++ foot-guns, you might want to skip to the bottom section for some new ideas in `fin`.

<br>

# The Code
Here's a small function that multiplies two numbers, and divides by 1024. To make the code *more* portable, we'll use `uint16_t` instead of `int`.

```c
// function to explore
uint16_t calc_1(uint16_t a, uint16_t b)
{
  return a * b / 1024;
}
```

This is the basic test code we will run on each platform:

```cpp
// test code
uint16_t a = 1000;
uint16_t b = 2000;
uint16_t result = calc_1(a, b);
```

If you do this math with a calculator, you'll get `1953.125` so we should be expecting `1953` due to integer truncation.

On the STM32, the result is `1953` as expected.

<br>

![image](https://github.com/fin-language/blog/assets/274012/d04df1ca-5591-4902-bb7b-7ba21461d5e2)


Maybe we even write some unit tests that run on our 64 bit computer and they agree with the STM32. We're good to go, right?


<br>

# Enter the AVR...

On the AVR, the result is `33`!

<br>

![image](https://github.com/fin-language/blog/assets/274012/9eca5e6f-131e-4ea1-8838-90c23b520027)

<br>

# What the flip!? `(╯°□°)╯︵ ┻━┻`
What's going on? We used fixed width integers! Why is the AVR program getting a different result than the STM32 program?

Well... the non portable `int` behavior snuck in without us noticing. In C, before arithmetic operations are performed, operands **smaller** than `int` are promoted to `int`.

## On 32+ bit int machines
This is essentially what our STM32 program does:

```c
// 32 bit int STM32 scenario
uint16_t calc_1(uint16_t a, uint16_t b)
{
  // `a * b`
  int promoted_a = a; // 32 bit int
  int promoted_b = b; // 32 bit int
  int interim_result = promoted_a * promoted_b;  // 32 bit int
  // interim_result = 0b00011110_10000100_10000000 (24 bits required)
  
  // divide by 1024 is like right shift 10 times
  interim_result = interim_result >> 10;
  // interim_result = 0b0111_10100001

  // C implicitly truncates the 24 bit result to 16 bits
  uint16_t result = (uint16_t)interim_result;
  return result;
}
```

The 32 bit `int` is large enough to not overflow the interim result, so we get the correct result of `1953`.

## On 16 bit int machines
However, when the operand is the same size or larger than the platform's `int`, there is no promotion.

This is basically what our AVR program does:
```c
// AVR scenario (16 bit int)
uint16_t calc_1(uint16_t a, uint16_t b)
{
  uint16_t interim_result = a * b; // 16 bit int. OVERFLOW!!!
  // can only hold lower 16 bits of ideal 24 bit result.
  // interim_result = 0b10000100_10000000 (33,920 dec)
  
  // divide by 1024 is like right shift 10 times
  interim_result = interim_result >> 10;
  // interim_result = 0b00100001 (33 dec)

  return interim_result;
}
```
<br>

Well fork me! We're not getting portable math just by using fixed width integers.

<br>
<br>

# So how do we get portable math in C?

## Cast your way to portable wrongness `⤜(⚆ᗜ⚆)⤏`
Many new C programmers cast absolutely everywhere even when not needed because they don't yet understand some of the tricky parts of C (promotions, conversions...). Casts seem to fix weird issues, so more is better. This is a bad habit because it makes the code harder to read/understand and can hide other issues.

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

Note that we don't need to wrap `(uint16_t)(a * b)` in parentheses like `((uint16_t)(a * b))` because the cast has higher precedence than the division. Adding extra parentheses is not the end of the world, but is also just extra noise.


## Widening Cast
We essentially need to widen the `a` and `b` operands to a type that is large enough to hold the interim result. In this case, we need to widen to `uint32_t` because the interim result may be up to 32 bits (16 bit x 16 bit).

```c
// returns the correct result!
uint16_t calc_4(uint16_t a, uint16_t b)
{
  return (uint32_t)a * b / 1024;
}
```

### Why not cast `b` as well?
Well, if you're on a 32 bit int machine... hold on to your butt...

Let's unpack what C is doing **implicitly** for us, and then we can talk about whether this is a good-thing™, or not.

```c
// when `int` is 32+ bits, this is what the compiler does:
uint16_t calc_4(uint16_t a, uint16_t b)
{
  // First, the `(uint32_t)a` cast happens as it has precedence
  // over multiplication.
  uint32_t cast_a = a;  // 32 bit

  // Second, `b` is promoted to `int` because b's uint16_t rank is
  // lower than the rank of `int` on a 32 bit system.
  int promoted_b = b;   // 32 bit

  // Third, `promoted_b` is converted to `uint32_t` to match the rank of `cast_a`.
  // Unsigned types have a higher rank than signed types.
  uint32_t converted_b = promoted_b; // 32 bit

  // Now we get to the multiplication. No surprises.
  // The operands are both `uint32_t`.
  uint32_t interim_result = cast_a * converted_b;  // 32 bit
  // interim_result = 0b00011110_10000100_10000000 (24 bits required)
  
  // The rank of `interim_result` carries through the calculation.
  // If 1024 was actually a uint16_t variable and not a literal, it would be
  // promoted/converted to match the rank of `interim_result`.

  // Divide by 1024 is like right shift 10 times.
  interim_result = interim_result >> 10;
  // interim_result = 0b111_10100001

  // C implicitly truncates to 16 bits
  uint16_t result = (uint16_t)interim_result;
  return result;
}
```

### On a 16 bit int machine
Casting `b` is still unnecessary on a 16 bit int machine.

```c
// when `int` is 16 bits (or less), this is what the compiler does:
uint16_t calc_4(uint16_t a, uint16_t b)
{
  // First, the `(uint32_t)a` cast happens as it has precedence
  // over multiplication.
  uint32_t cast_a = a;  // 32 bit

  // `b` is NOT promoted to `int` because b's uint16_t rank is
  // higher than the rank of `int` on a 16 bit int system.
  // Unsigned types have a higher rank than signed types.

  // Instead, `uint16_t b` is converted to `uint32_t` to match the rank of 
  // `uint32_t cast_a`.
  uint32_t converted_b = b; // 32 bit

  // Now we get to the multiplication. No surprises.
  // The operands are both `uint32_t`.
  uint32_t interim_result = cast_a * converted_b;  // 32 bit
  // interim_result = 0b00011110_10000100_10000000 (24 bits required)
  
  // ... same as 32 bit int example below here ...

  // The rank of `interim_result` carries through the calculation.
  // If 1024 was actually a uint16_t variable and not a literal, it would be
  // promoted/converted to match the rank of `interim_result`.

  // Divide by 1024 is like right shift 10 times.
  interim_result = interim_result >> 10;
  // interim_result = 0b111_10100001

  // C implicitly truncates to 16 bits
  uint16_t result = (uint16_t)interim_result;
  return result;
}
```

## 📜 There are more rules... yay! 😒
Resources in order of increasing complexity/detail:

* https://user-web.icecube.wisc.edu/~dglo/c_class/promo_conv.html
* https://wiki.sei.cmu.edu/confluence/display/c/INT02-C.+Understand+integer+conversion+rules
* https://en.cppreference.com/w/c/language/conversion


<br>
<br>

# Casts can hide mistakes
C casts are very powerful, but need to be treated with care as they can easily hide mistakes. This usually isn't a big deal if you are dual targeting your code for both embedded and a PC for unit testing. However, if you are relying solely on the embedded compiler (no unit tests), then you really need to be extra careful.

If for some reason we refactored our function to instead take a `pointer` to a `uint16_t`, we might make a mistake like below. Many compilers (even expensive ones like IAR) will not warn about this because we've told the compiler that we know what we're doing with the cast.


```c
// Before refactor to take a pointer
uint16_t calc_4(uint16_t a, uint16_t b)
{
  return (uint32_t)a * b / 1024;
}
```

```c
// Oops!
// After refactor to take a pointer.
uint16_t calc_5(uint16_t *a, uint16_t b)
{
  // original cast to widen `a` is still there,
  // but now it is hiding a mistake.
  return (uint32_t)a * b / 1024;
}
```

Should a developer catch that mistake while refactoring? Yes. Will they? Usually, but not always. Deadline pressure, multitasking and other distractions (manager popping by for a visit) make it easy to miss things like this.

You might get lucky and have the compiler flag this if your embedded system's pointer size is different than int, but that usually isn't the case on embedded systems.
> *GCC warning on 64 bit system: cast from pointer to integer of different size*

Without unit testing or getting lucky, that simple mistake can result in hours of debugging.

`(ノಠ益ಠ)ノ彡┻━┻ arrrgg`

## A safer way to widen
A safer way to widen an integer is with a tiny helper function.

```c
// This function call is optimized away
// when any optimization level is enabled.
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
> Note! `-Wconversion` is not enabled in `-Wall` or `-Wextra` on GCC. You need to add it explicitly.

The version of IAR (💰💰💰) I use at work has nothing comparable to GCC's `-Wconversion` checking unfortunately. Another good reason to write unit tests that run on a PC and use a different compiler than your embedded compiler. Different compilers have different strengths and weaknesses.

With `-Wconversion`, GCC will warn about misusing the widening function.

```c
uint8_t u8 = ...;
uint16_t u16 = ...;
uint32_t u32 = ...;
int32_t i16 = ...;
int32_t i32 = ...;
float f32 = ...;
uint32_t wider;

// ok - no warning
wider = widen_to_u32(u8);
wider = widen_to_u32(u16);
wider = widen_to_u32(u32);

// warnings:
wider = widen_to_u32(&u8); // expected ‘uint32_t’ but argument is of type ‘uint8_t *’
wider = widen_to_u32(i16); // conversion to ‘uint32_t’ from ‘int32_t’ may change sign of the result
wider = widen_to_u32(i32); // conversion to ‘uint32_t’ from ‘int32_t’ may change sign of the result
wider = widen_to_u32(f32); // conversion to ‘uint32_t’ from ‘float’ may change its value
```

> Play with online: https://rextester.com/VSVI53810

We now need to be explicit with our truncation from 32 bits to 16 bits for the result:

```c
uint16_t calc_7(uint16_t a, uint16_t b)
{
  return (uint16_t)(widen_to_u32(a) * b / 1024);
}
```

```c
// or if you want to avoid potential cast issues, you can
// also make a safer helper function that will be optimized away.
static inline uint16_t trunc_u32_to_u16(uint32_t a)
{
    return (uint16_t)a;
}

uint16_t calc_7b(uint16_t a, uint16_t b)
{
  return trunc_u32_to_u16(widen_to_u32(a) * b / 1024);
}
```

Creating and maintaining all these safer helper functions is a bit of a pain, and does start to
obscure the math/algorithm.

If safety is a priority, it can be worth it though.

<br>
<br>



# C/C++ mixing signed and (gasp!) unsigned 🙀
First a touch of history

> The guy who created Java thinks that programmers aren't smart enough to handle mixing unsigned and signed numbers so Java doesn't have unsigned numbers. I mean, mistakes were made, OK many mistakes were made, but don't take my unsigned integers!
>
> C# handles this well. We get unsigned integers and **none** of the C/C++ foot-guns related to mixing signed/unsigned.

Back to our regularly scheduled programming...

Here's some more non-portable code that you might see in the wild.

```c
// implementation defined behavior in C
uint16_t a = 1000;
int16_t b = -1;
bool a_is_greater = a > b; // ???
```

What is the result of `a_is_greater`? Well it depends on the platform.

Our AVR gives us `false`, while our STM32 gives us `true`.

## What!? Why? That code looked so simple and pure 🦄
The solution to this problem is really easy. Just throw out all your Arduino's. Problem solver :)

C and C++ are really old languages and this one particular decision made sense way back in the day, but it didn't age well.

When comparing signed and unsigned integers with size >= `size(int)`, the signed integer is converted to unsigned. Yep, a negative signed number can end up looking like a **HUGE** unsigned number.

This behavior is also non-portable because it relies on promotions to int (which are implementation defined). This explains why our AVR and STM32 give different results.

<!-- This page has lots of good info on C promotions and conversions: https://www.geeksforgeeks.org/type-conversion-c/ -->

## C/C++ Solution - Widen Operands First
Luckily, many compilers are pretty good at warning about this if you enable it (too bad it isn't a default). You can enable it with `-Wsign-compare` on GCC.

One solution is to manually widen the signed and unsigned operands to something that can represent both of them.

```c
uint16_t a = 1000;
int16_t b = -1;
bool a_is_greater = (int32_t)a > (int32_t)b; // true everywhere
```
Or you can just widen one side and let implicit rank conversions handle the other. All the typical issues with casting can still occur. You could create a widening function like shown earlier to be safer.


## C/C++ Solution - Test Signed
If we don't want to widen the operands to something that can hold both signed and unsigned values, we can test the signed integer for negative first and then compare both as unsigned.

```c
uint16_t a = 1000;
int16_t b = -1;
bool a_is_greater = b < 0 || a > (uint16_t)b;
```

Again, watch out for future refactoring or copy paste mistakes where the size of `b` is changed, but the unsigned cast is not. The cast is not actually required, but many people put it in to be explicit.


<br>
<br>

# Feeling Confused? 🤯
Understandably. This can be a lot to take in if this is your first time coming across it.

The good news is that lots of modern languages solve these problems.

<br>
<br>


# Simpler With Fin
Many people find the above C/C++ foot-guns confusing. And even if you do understand it, it still takes time to explain to others and catch in code reviews...

Fin code is much easier to read and understand:
* u16 types use u16 math...
* fin math is portable across platforms by default.
* if you need more width, be explicit.
* there are no implicit promotions to `int`.
* there are no implicit truncations.
* it prevents undefined behavior.

## fin - safe mixing signed and unsigned
In `fin`, this is a non-issue. Nice and simple.

```c
// fin - fully defined and expected behavior
u16 a = 1000;
i16 b = -1;
bool a_is_greater = a > b; // true! "Look ma! No casts or implementation defined behavior!"
```

## fin - no implicit promotion to `int`
C code promoting to a system defined `int` results in non portable code, is confusing at times and can lead to unexpected results.

Our `fin` code works as written (no confusing implicit C/C++ behavior) and runs the same on any embedded platform regardless of CPU architecture/size.

If you want something to be widened to a larger size in fin, you simply need to ask for it.

```cs
// fin - potential syntax
u16 a = /* something */;
u32 temp = (u32)a;  // fin casts are much safer than C casts
u32 temp = a.u32;
u32 temp = a.u32();
u32 temp = a.to_u32();
u32 temp = a.widen_u32();
u32 temp = a.up_u32();
```

## fin - no implicit truncation
C (without extra, extra, extra warnings) will happily truncate your data without telling you. Often this is what we want, but sometimes this does bite us. It can lead to unexpected results, undefined behavior, and bugs.

In fin, we need to be explicit about narrowing conversions.

We need to clarify our intentions and be explicit:
1. **mod/wrap**: we want a u32 to be wrapped/truncated to a smaller type (like u16).
    * **mod**ulus like conversion
    * we don't care if we lose any info in the top 16 bits.
2. **as**: we want a u32's value `as` a smaller data type.
    * if the value won't fit, we want to know about it.
3. **saturate/clamp**: we want a u32's value to be clamped to a smaller type (u16).
    * if the value won't fit, it should be clamped to the new types min/max.

I'm still working on the syntax for this, but it may look something like below.

Similar conversions exist for signed types and converting between signed/unsigned.

### `mod/wrap` conversions
```cs
// fin - mod/wrap potential syntax:
u32 temp = ...;
u16 result = temp.mod_u16();  // modulus like conversion
u16 result = temp.wrap_u16(); // modulus like conversion
u16 result = unchecked((u16)temp); // modulus behavior and prevents any error checking
```

### `as` conversions
```cs
// fin - 'as' potential syntax:
u32 temp = ...;
u16 result = temp.as_u16(); // error if doesn't fit
u16 result = (u16)temp;     // error if doesn't fit
```

### `saturate/clamp` conversions
```cs
// fin - 'clamp' potential syntax:
u32 temp = ...;
u16 result = temp.sat_u16();    // saturates if too large
u16 result = temp.clamp_u16();  // saturates if too large
```



## fin - trapping overflow/underflow
In C, signed overflow/underflow is undefined behavior. This means that the compiler can do whatever it wants. It can even make demons fly out of your nose. Copilot generated that last sentence, but it is mostly true :) [See here](https://blog.regehr.org/archives/213).

In fin, overflow/underflow is caught by default. This is a good thing. It is much better to know that you've done something wrong than to have the compiler do whatever it wants.

You can also use helper functions to prevent undefined behavior.

### Saturating methods
```cs
// fin potential syntax
u16 result = a.sat_add(b);  // a + b
u16 result = a.sat_mult(b); // a * b
```

### Sticky error tracking

```cs
// "sticky" error tracking
StickyErr err = /* stack allocate */;
result = a.add(b, err);
result = result.add(x, err).add(y, err); // can chain
result.inc(err);
// check err after all are completed to see if any error occurred.
```

We can also make sticky error handling a bit nicer in bulk. Something like this:

```cs
u16 result;
StickyErr err = Math.capture_errors(() => {
    result = a + b;
    result += x + y;
    result++;
});
```

### Special types

Sometimes you want modulus/wrapping or saturating behavior on overflow though. To make that convenient, we will have different types (example just for `i8`):

| Type  | Overflow Behavior<br>(on target hardware) | Overflow Behavior<br>(sim/testing) | Notes                         |
| :---- | :---------------------------------------- | :--------------------------------- | ----------------------------- |
| `i8`  | error\*                                   | exception with trace               | Suggested type                |
| `i8m` | modulus wrapping                          | modulus wrapping                   | Good for some algorithms      |
| `i8s` | saturates at limit                        | saturates at limit                 | Good for control systems      |
| `i8c` | undefined C behavior                      | exception with trace               | When you are sure & speed\*\* |
> same for `u8`, `u8m`, `u8s`, ...

* \* Not exactly sure how errors will present yet...
* \*\* It's generally advised to use `i8` instead of `i8c` and use a safety escape when speed is required. This makes your code safe by default when shared around. It might be fine in your particular situation, but another dev might copy your pattern for a situation where it isn't actually safe. This happens all the time in C...

In the future, we will be able to add guards that fin can use to determine that there is no chance of under/overflow. In these cases, fin will omit any `i8` runtime checks on target embedded hardware.

<br>

# Panic or exception?
Rust and zig both panic (AKA kill the program) when they detect an integer overflow/underflow. In fast/release builds, they don't check for overflow/underflow at all. Many people dislike these decisions (especially not checking), but runtime speed is an primary goal of both rust and zig.
> Fin will have more granular methods of disabling safety checks *without* modifying source code. This prevents copy paste errors as detailed above in \*\*.

Panicking makes sense for the needs of zig and rust, but I think it is a terrible default for embedded. Embedded systems often need to be reliable and may need to keep running in a "limp" mode even if a bug occurs.

## 💀 Killing the program should <u>**NOT**</u> be the default...💀
... at least for embedded.

If my code is controlling a tractor and a bug in a data logging system allows a variable to overflow, I want the program to have the choice on how to react:
* I might want to shutdown the tractor gracefully (not just reboot and slam to a halt).
* I might want to send a warning to the user, disable the data logging system, and keep running as the data logging system is not crucial to operation.
* I might want to restart the data logging system and keep running.
* ...

I'd like to explore more options before resorting to panic or exceptions.

Hidden flow control is not ideal!

## Why do rust/zig panic?
Because having error handling absolutely everywhere for any math (like `i++` in a for loop) is a pain. Rust and zig don't have exceptions, and requiring users to check all math in a rust/zig way is too verbose.

Fin will support lightweight exceptions (not heavy like C++), but they will also be very <u>**visible**</u>.

When I'm working on a system that needs to be reliable, I want to see all the places where exceptions can occur at a glance. I don't want to have to dig through code and method declarations to find them.

More thoughts to come...



<br>

# Wait! I forgot...
With all the talk about the various foot-guns in C/C++, I forgot to show a solution to our original problem using fin.

Here's one possible solution that saturates if the result is too large.

```cs
// fin - working solution
u16 calc_1(u16 a, u16 b)
{
  return (a.u32 * b / 1024).sat_u16();
}
```

I don't know about you, but I really like it :) Easy to read, portable, and safe.
