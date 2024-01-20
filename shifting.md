# Shifting
In `fin`, only unsigned numbers can be shifted.

Right shifts `>>` are generally safe (except when shifting by an illegal amount).

Left shifts `<<` can overflow. Sometimes we care about that and, sometimes we don't.

Like many modern languages, fin has operations for wrapping shifts (overflow is OK) and regular shifts (overflow is an error).

## When overflow is expected and OK - `wrap_lshift()`
When we don't care if the shift overflows, we want *wrapping* behavior.

This is common in CRC, LFSR, and hashing algorithms. Outside of these types of algorithms, I don't see wrapping shifts used very often.

```c
// c
if (crc & 0x80000000) {
    crc = (crc << 1);
    crc = crc ^ SOME_POLYNOMIAL;
} else {
    crc = (crc << 1);
}
```

```c
// fin
if (crc & 0x80000000) {
    crc = crc.wrap_lshift(1);
    crc = crc ^ SOME_POLYNOMIAL;
} else {
    crc = crc.wrap_lshift(1);
}
```

Even though we don't care about overflows, we still want to guard against undefined C behavior.

### Errors
While we don't care about overflows for `wrap_lshift()`, we still want to guard against undefined C behavior:
* shifting by a negative value
* shifting by more than the type can hold

Because of this, you either need to have `math.unsafe_mode()` or `math.capture_errors(err)` active.

In unsafe mode, errors will be caught during C# simulation, but not at C runtime.


## When we don't expect overflows - `<<`
In many cases, we don't want or expect overflows. We want to know if we are shifting by too much as it will usually reveal a programming mistake. This is very common when assembling packet/binary data. Bit fields (in C) can sometimes be used for this, but bitfield ordering and layout isn't actually portable. Shifting and masks - this is the way.

In the embedded code that I regularly work in, I see a lot more of this type of "regular" shifting over wrapping shifts.

```c
transmit_message |= (rx_message.source_address << 21); // bug if shifted source_address overflows
```

```c
interrupt_source |= (1 << source); // bug if overflows
```

### Errors
In addition catching overflows, we also want to guard against undefined C behavior:
* shift causing an overflow
* shifting by a negative value
* shifting by more than the type can hold

Because of this, you either need to have `math.unsafe_mode()` or `math.capture_errors(err)` active.

In unsafe mode, errors will be caught during C# simulation, but not at C runtime.
