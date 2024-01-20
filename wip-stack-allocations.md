originally had thought about something like 

# `mem.place<type>`
```cs
using var bike = mem.place<Bike>();
Bike bike = mem.place<Bike>();
```

# `mem.place(new Bike())`
Needed if we want to pass arguments to the constructor.

```cs
using var bike = mem.place(new Bike(12));
```


# Scoped Allocators?
Makes me a bit uneasy to be honest. Looks nice though. Allows easy nesting.

```cs
void my_func()
{
    mem.default_stack_allocate(); // affects "new ()"

    using var bike = new Bike(12);

    // allows nesting
    using var bike2 = new Bike(12, new Wheel());
}
```

# Ditch `using`
Now that we have Fody to keep track of scopes, we almost don't need `using` anymore. We just need it in inner function blocks.

```cs
void my_func(u8 a)
{
    mem.default_stack_allocate(); // affects "new ()"

    Bike bike = new(12);

    if (a > 12)
    {
        Bike temp = new(a);
        //...
    } // temp needs to be destroyed here
}
```

I think for now, we ditch `using`. We aren't even tracking bad memory usage right now anyway.
When we want that, we can probably create a Fody plugin that does it.


# `mem.stack()`
```cs
Bike bike = mem.stack(new Bike());
```


# stack alloc by default
Fin is intended for embedded systems. We could make stack allocations easy to write as they are used the most.
Heap allocations could require more work.

```cs
Bike bike = new Bike(); // on stack
Bike bike2 = new();     // on stack

Bike? heap_bike = mem.malloc(new Bike()); // requires ptr type because it could fail
Bike? heap_bike_2 = mem.heap(new Bike());

mem.free(heap_bike);
```

```cs
Bike? bike = mem_pool.allocate(new bike()); // null on err

Packet ack = mem.stack(new Packet()); // and heap

```

