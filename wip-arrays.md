
```cs
array_s8<u8> nums = ///...
mem.capture_errors(err);

nums[5] = 0;  // may set errors implicitly

// explicit error handling
nums.set(5, 0, err);  // may set errors
nums.get(5, err);  // may set errors


array8<u8> nums;
s8_array<u8> nums;

array_of_8<Car> cars;
arr8<Car> cars;

arr_of_8<Car> cars;


```


Apart from naked C style arrays that do not track length, we can also have arrays that do track their length.

`arr8<T>` is an of type `T` with a `u8` to track size.

We can either implement this as a struct with a pointer to the data or with a flexible array member (FAM).

The FAM approach is more efficient (memory & speed), but it has some restrictions.

```c
//fin: `arr8<i16>`
typedef struct arr8_i16 {
    uint8_t size;
    int16_t elements[]; // C99 flexible array member (FAM)
} arr8_i16;
```

If we use FAMs, we have the following restrictions:

- The incomplete array type must be the last element within the structure.
- There cannot be an array of structures that contain a flexible array member.
- Structures that contain a flexible array member cannot be used as a member of another structure.
- The structure must contain at least one named member in addition to the flexible array member.
- https://wiki.sei.cmu.edu/confluence/display/c/DCL38-C.+Use+the+correct+syntax+when+declaring+a+flexible+array+member

`arr8_i16` cannot exist, but it can be used as a pointer.

## How to allocate memory for FAM?

### alloca()
```c
// c99
void some_func(void)
{
    // fin: var numbers = mem.stack(new arr8<i16>(10));
    arr8_i16 *numbers = alloca(sizeof(arr8_i16) + 10 * sizeof(i16)); // might include some padding
    arr8_i16_ctor(numbers, 10); // zero initializes array elements and sets size
}
```

Note: this won't work for global file scope variables though. Can't `alloca()` there.

### unique definition
```c
// c99 header somewhere
typedef struct arr8_i16_size10 {
    uint8_t size;
    int16_t data[10];
} arr8_i16_size10;

// c99
void some_func(void)
{
    // fin: var numbers = mem.stack(new arr8<i16>(10));
    arr8_i16 *numbers = (arr8_i16*)&(arr8_i16_size10){ .size = 10 };
}
```

A bit more tricky because the the definition for `arr8_i16_size10` needs to go somewhere, but it is also potentially more efficient (no ctor call).

Might also violate strict aliasing rules, but I'm not 100% sure. There is no chance of aliasing, but it is type punning. It compiles and runs fine with GCC: https://rextester.com/UJWD7601


# Slices
Slice naming comes from rust. C++ has array_view that is readonly and span that is readwrite.

slice8<T> is an array of type T with a u8 to track size and a pointer to the data.

```c
//fin: `slice8<i16>`
typedef struct slice8_i16 {
    uint8_t size;
    int16_t * elements; // can point to middle of an array
} slice8_i16;
```

