# brain dump
this stuff will likely change


# main points
- no hidden flow control unless asked for
- allows chaining
- forces check of error argument
- can defer check of error argument


# no hidden flow control unless asked for

```cs
u16 calc_with_widen(u16 a, u16 b, Err err) {
    u16 result = (a.u32 * b / 1024).to_u16(err);    // multiplication can't overflow because of increase to u32 beforehand
    return result;
}
```

```cs
u16 calc_no_widen(u16 a, u16 b, Err err) {
    // u16 result = (a * b / 1024);     // mult can overflow. can't use regular `a * b`. have to use `a.mult(b, err)`
    u16 result = a.mult(b, err) / 1024;
    return result;
}
```

```cs
u16 calc_saturate(u16 a, u16 b) { // no error argument
    u16 result = a.sat_mult(b) / 1024;
    return result;
}
```

/////////////////////////////////////////////////////////

```cs
u16 long_calc_with_widen(u16 a, u16 b, Err err) {
    using Scope s = new(err); // catches errors
    
    u32 index = a.u32 * b / c + obj.get_offset();
    if (err) return 0;  // or can call `err.defer_check()` and check later

    arr[index] = arr[index-1] + arr[index-2];
    if (err) return 0;

    return result;
}

u16 get_offset() {
    using Scope s = new(); // required for simulation so that C# runtime knows scope rules
    u16 offset = x.sat_add(y);
    return offset;
}
```

```cs

```
