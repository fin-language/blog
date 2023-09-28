# Conflicting Desires for Error Handling
* We don't want hidden flow control
    - like exceptions (especially bubbling up multiple levels)
* We don't want to have to check for errors everywhere
    - in arithmetic
    - for every memory access


<br>

# No-Panic Embedded
Any program that panics has hidden flow control. Panics are just exceptions that can't be recovered from. Hidden flow control is bad for reliable systems.

One goal of fin is to never-ever panic unless explicitly forced by the programmer using external C code. Fin code will not hide panics in libraries or innocent looking syntax like `a + b` or `c = arr[i]`. Fin will still however detect issues. It just doesn't panic.

A number of languages like Rust and zig panic (AKA kill the program) when they detect an integer overflow/underflow. In fast/release builds, they don't check for overflow at all. Many people dislike these decisions (especially not checking).

Part of the rust rationale for not checking for overflow in release builds is that memory accesses are still checked so security issues are greatly reduced. That doesn't help with control systems or algorithms that need to be reliable though.

And while it is good that they prevent bad memory access, it is another source of hidden flow control in rust/zig.

Panicking makes sense for the primary needs of zig and rust, but I think it is a terrible default for embedded. Embedded systems often need to be reliable and may need to keep running in a "limp" mode even if a bug occurs.

Fin code never panics and removes a lot of hidden flow control in other languages. This is a good thing. It allows programmers to have more control of their software and feel more at ease. They don't need to be quite so paranoid about every single line of code and how it might kill their program.

However, it is important to note that this doesn't mean that our fin program is bullet proof. Our program can still crash if our code angers the pixies inside a processor (misuse of registers, peripherals, ...).



## Examples of Where Panicking is Bad

### Error In Optional Subsystem
If my code is controlling a tractor and a bug in a data logging subsystem allows a variable to overflow, I want the program to have the choice on how to react:
* I might want to shutdown the tractor gracefully (not just reboot and slam to a halt).
* I might want to send a warning to the user, disable the data logging system, and keep running as the data logging system is not crucial to operation.
* I might want to restart the data logging system and keep running.
* ...

### Sensor Glitch
You might be programming a quad copter controller that is constantly taking sensor readings and adjusting the motors. Sometimes sensors can glitch or be excessively noisy. If a loose wire, vibration, transient, or EMI cause a sensor reading glitch, we don't want the quad copter to start falling out of the sky while the processor panics and reboots. We want the program to have the choice on how to react:
* I might want to ignore the bad reading and keep running for a few more iterations to see if it recovers.
* I might want to lower the quad copter gracefully to the ground.
* I might want to send a warning to the user and keep running.
* ...

Should the developer have taken this into account? Yes, but it is an easy thing to overlook. Especially for less experienced/paranoid developers. All the readings look sane on the bench, so they code to what they expect to see.

### Inside a Kernel
Linus Torvalds:
> _With the main point of Rust being safety, there is no way I will ever accept "panic dynamically"..._
>
> _I do think that the "run-time failure panic" is a fundamental issue._
>
> _Because kernel code is different from random user-space system tools.
> Running out of memory simply MUST NOT cause an abort.  It needs to
> just result in an error return._
> <br>https://lkml.org/lkml/2021/4/14/1099

## More Reading
https://github.com/rust-embedded/wg/issues/551



<br>
<br>
<br>


---
---
---
# BRAIN DUMP AND WIP BELOW
---
---
---

<br>
<br>
<br>

# Memory Access (int array)

```cs
array<int> arr = // ... [0:10, 1:20, 2:0]
arr[i] = arr[i-1] + arr[i-2];
// err check required here by fin compiler. Can also defer check for speed.
Err.defer();
```

if `arr[i-2]` is invalid, what does fin do and return?
* return 0, set Err

* `arr[1] = arr[0] + arr[-1];`
* `arr[1] = arr[0] + 0;`

What about `arr[0]`? Does it return a real value or check Err and return 0?
* I think return real value.
* `arr[1] = 10 + 0;`

Do we store `arr[1]` knowing that there was an error?
* not sure. Might need user to indicate what they want.
* checking for err first makes things slower.
* feel safer to prevent writes to memory if there was an error.
* it is easy for people to prevent writes to memory if they want to though:

```cs
array<int> arr = // ... [0:10, 1:20, 2:0]
var temp = arr[i-1] + arr[i-2];

if (Err)
    return 0; // leaves error set

arr[i] = temp;
```

If we return null
```cs
array<int> arr = // ... [0:10, 1:20, 2:0]
int? temp = arr[i-1] + arr[i-2];

if (temp == null || Err)
    return 0; // leaves error set

arr[i] = temp.Value;
```



# Memory Access (object array)
What we would like to write:
```cs
array<Car> cars = // array of car references
cars[i].fuel = cars[i-1].fuel + cars[i-2].fuel;
// err check required here by fin compiler. Can also defer check for speed.
```

if `cars[i-2]` is invalid, what does fin do and return?
* sets err.
* returning null might be best.

```cs
void Test2(Car? a, Car? b, Car? c)
{
    if (c != null)
        c.fuel = (a?.fuel + b?.fuel) ?? 0;
}
```


```cs
array<Car> cars = // array of car references
Car? car = cars[i];  // returns null on fail and sets 
if (car == null || Err)
    return 0;

car.fuel = (cars[i-1]?.fuel + cars[i-2]?.fuel) ?? 0;
// err check required here by fin compiler. Can also defer check for speed.
```


Using Ternary:
- Slower. Repeated access. Very verbose.
```c
// car.fuel = (cars[i-1]?.fuel + cars[i-2]?.fuel) ?? 0;
car->fuel = (array_of_Car__get(cars, i-1, err) ? array_of_Car__get(cars, i-1, err)->fuel : 0) + (array_of_Car__get(cars, i-1, err) ? array_of_Car__get(cars, i-1, err)->fuel : 0);
```

```c
// car.fuel = (cars[i-1]?.fuel + cars[i-2]?.fuel) ?? 0;
Car* car_i_1 = array_of_Car__get(cars, i-1, err);
Car* car_i_2 = array_of_Car__get(cars, i-2, err);

if (car_i_1 == null || car_i_2 == null)
    car->fuel = 0;

car->fuel = car_i_1->fuel + car_i_2->fuel;
```


```cs
array<Car> cars = // array of car references
int? fuel = cars[i-1]?.fuel + cars[i-2]?.fuel;
Err.defer_check();  // check for error later

Car? car = cars[i];  // returns null on fail and sets 
if (car == null || Err)
    return 0;

car.fuel = fuel.Value;
```

## Simulation Returns Dummy Object
if any car doesn't exist, C# returns dummy and sets Err. We need C and C# results/behavior to match.

Could be a problem if a function is called on dummy object. Maybe we throw on simulation if dummy object has method called.

Downside is that generated C doesn't look like C# as much. It has a bunch of if statements.

```cs
array<Car> cars = // array of car references
cars[i].fuel = cars[i-1].fuel + cars[i-2].fuel; // don't return null
// err check required here by fin compiler. Can also defer check for speed.
```

```c
Car* car_i_1 = array_of_Car__get(cars, i-1, err); // returns null
//...
```

## Everything Returns Dummy Object
Would require every class to have a dummy object. It could be const though if we prevent writes to dummies.

We need C and C# results/behavior to match.

```cs
array<Car> cars = // array of car references
cars[i].fuel = cars[i-1].fuel + cars[i-2].fuel; // don't return null
// err check required here by fin compiler. Can also defer check for speed.
```

```c
// cars[i].fuel = cars[i-1].fuel + cars[i-2].fuel;
array_of_Car__get(cars, i, err)->fuel = array_of_Car__get(cars, i-1, err)->fuel + array_of_Car__get(cars, i-2, err)->fuel;
```

```c
// with overflow checks
// cars[i].fuel = cars[i-1].fuel + cars[i-2].fuel;
array_of_Car__get(cars, i, err)->fuel = array_of_Car__get(cars, u8_sub(i, 1), err)->fuel + array_of_Car__get(cars, u8_sub(i, 2), err)->fuel;
```

```c
// with overflow checks
// cars[i].fuel = cars[i-1].fuel + cars[i-2].fuel;
Cars* car_i = array_of_Car__get(cars, i, err); // cars[i]
Cars* car_i_m1 = array_of_Car__get(cars, u8_sub(i, 1), err);  // cars[i-1]
Cars* car_i_m2 = array_of_Car__get(cars, u8_sub(i, 2), err);  // cars[i-2]
car_i->fuel = car_i_1->fuel + car_i_2->fuel;
```

### Reduce RAM Usage
Make a union of all dummy types. This will save a lot of RAM, but it still might not be enough. What if a structure has 1 KB array? A union won't really help then. We need to prevent writes to dummies.

Could ask user to move arrays outside of class. Not ideal though.

### Prevent Writes to Dummies
```c
// cars[i].fuel = cars[i-1].fuel + cars[i-2].fuel;
{ // scope for temporary vars
    // with overflow checks
    Cars* car_i = array_of_Car__get(cars, i, err); // cars[i]. real or null.
    Cars* car_i_m1 = array_of_Car__get(cars, u8_sub(i, 1), err);  // cars[i-1]. real or dummy garbage.
    Cars* car_i_m2 = array_of_Car__get(cars, u8_sub(i, 2), err);  // cars[i-2]. real or dummy garbage.

    if (car_i != NULL && !err.active)
        car_i->fuel = car_i_1->fuel + car_i_2->fuel;
}

// dummies may point to real memory used by other objects. If we allow storing data read out of dummies, it could be a potential security issue (reading keys). If we don't allow storing data out of dummies, then it could be OK. Where security is more of a concern, we often have more RAM and could just read from blank dummies instead of real memory.
```


### Nulls Instead of Dummies
```c
// cars[i].fuel = cars[i-1].fuel + cars[i-2].fuel;
{ // scope for temporary vars
    // with overflow checks
    Cars* car_i = array_of_Car__get(cars, i, err); // cars[i]. real or null.
    Cars* car_i_m1 = array_of_Car__get(cars, u8_sub(i, 1), err);  // cars[i-1]. real or null.
    Cars* car_i_m2 = array_of_Car__get(cars, u8_sub(i, 2), err);  // cars[i-2]. real or null.

    if (car_i != NULL && car_i_m1 != NULL && car_i_m2 != NULL && !err.active)
        car_i->fuel = car_i_1->fuel + car_i_2->fuel;
}

// dummies may point to real memory used by other objects. If we allow storing data read out of dummies, it could be a potential security issue (reading keys). 
// If we don't allow storing data out of dummies, then it could be OK. Where security is more of a concern, we often have more RAM and could just read from blank dummies instead of real memory.
```

# Does `cars[i-1]` set err or just return NULL?
It seems like it should just return NULL and not set Err. This allows more explicit handling for users that want it.

We could make dummy access more convenient though:
```cs
cars.get_d(i).fuel = cars.get_d(i-1).fuel + cars.get_d(i-2).fuel;
```

# Or we could have `cars[i]` be non-null dummy
I think I prefer this. Keep the default syntax simpler and check for Err afterwards.

If you want explicit error handling with NULLs, you can write:
```cs
int? fuel = cars.get_n(i-1)?.fuel + cars.get_d(i-2)?.fuel;
```





# Approaches to Error Handling
* Out parameters `int foo(int a, int b, Err* err)`
    - potentially slower than return value
    * Sticky Errors
* Return codes
    - prevents chaining
    - sentinel values (use -128 or 255 to mean error)
    - impossible for constructors
* Optional/nullable types
* Errno (global error code)
* Undefined behavior
* -----
* error callback handler
* Panic (kill program)
* setjmp/longjmp
    - unclear flow/control
* Exceptions

# What we want
* default should be to detect integer overflows. not sure what it looks like.
* panic on integer overflow is not a good approach for embedded control systems.
* we don't want super verbose code that checks for overflows everywhere.
* some code might want to use exceptions (when they are supported), but most will not.
* some code might want to use 


# Review of other languages

## Rust
Integer overflows panic (aka kill the program) in debug builds. No checking in release builds.

Writing code that is safe from overflows in Rust ends up being very verbose:

```rust
fn calc_7(a: u16, b: u16) -> Option<u16> {
    match (u32::from(a)).checked_mul(u32::from(b)) {
        Some(result) => match result.checked_div(1024) {
            Some(result) => u16::try_from(result).ok(),
            None => None,
        },
        None => None,
    }
}

fn main() {
    let a: u16 = 1000;
    let b: u16 = 2000;
    
    match calc_7(a, b) {
        Some(result) => println!("Result: {}", result),
        None => println!("Overflow occurred"),
    }
}
```



# C# Hindsight

> Should C# checked arithmetic have been the default?
>
> I understand why the desire was there to make unchecked arithmetic the default: it’s familiar, it’s faster, a new language is going to be judged in part on benchmarks, and so on. But with hindsight, I would rather that checked arithmetic have been the default, and users be forced to turn it off for precisely those situations where the inner-loop performance is genuinely impacted by this nano-optimization. We have other safety features like array bounds checking on by default; it makes sense to me that arithmetic bounds checking would be on by default as well. But again, we’re stuck with it now.
>
> https://web.archive.org/web/20150415222821/https://ericlippert.com/2015/04/13/what-is-the-unchecked-keyword-good-for-part-two/




