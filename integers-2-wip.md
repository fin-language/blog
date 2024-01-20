
# Farther Future Error Handling
Needs more thought, but I'm exploring the idea of having a global `Err` object. Checking `Err` would be required so that users don't accidentally forget (like `errno`). This would allow us to write code like below (simple, easy to read, no hidden exceptions), but also be perfectly safe. No runtime panics either like Rust.

```cs
// fin
u8 calc_stuff(u16 a, u16 b, u16 c)
{
  return ((a + b) / c).to_u8();
}
```

You have to either check `Err` or defer the check.

```cs
// verbose explicit option (other options follow)
u8 calc_duty_cycle(u16 a, u16 b)
{
  u8 result = calc_stuff(a, b, 2);
  if (Err.is_active) return 0;  // required error check

  result = do_some_other_stuff(result);
  if (Err.is_active) return 0;  // required error check

  err++; // can also set Err flag
  if (Err.is_active) return 0;  // required error check

  return result;
}
```

```cs
// equivalent to explicit version, but much more succinct
u8 calc_duty_cycle(u16 a, u16 b)
{
  u8 result = 0;
  
  break_on_err(() => {
    // any error in this block will set Err flag and break out of `break_on_err`
    result = calc_stuff(a, b, 2);
    result = do_some_other_stuff(result);
    err++;
  });

  return result;
}
```

```cs
// Verbose explicit version that defers error checking for speed.
// The caller still has to check or defer the check.
u8 calc_duty_cycle(u16 a, u16 b)
{
  u8 result = calc_stuff(a, b, 2);
  Err.defer_check();  // satisfies Fin

  result = do_some_other_stuff(result);
  Err.defer_check();  // satisfies Fin

  err++; // can also set Err flag
  Err.defer_check();  // satisfies Fin

  return result;
}
```

How the global `Err` object is implemented in generated C code will be user configurable. It can either be passed as an argument to C functions (the default), or rely on some kind of user provided global variable that is thread safe.

<br>
<br>
<br>