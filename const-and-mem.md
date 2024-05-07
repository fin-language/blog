
# `c_const<T>`

### parameter: constant object
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<table>
<tr><td>fin</td><td>C99</td></tr>
<tr><td>

```csharp
void use_bike(c_const<Bike> bike)
{
    //...
}
```
</td><td>

```c
void use_bike(const Bike * bike)
{
    //...
}
```
</td></tr>
</table>
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->

This feels a bit weird because a fin `Bike` is like a C pointer to Bike (`Bike*`) so `c_const<Bike>` feels like it would be a const pointer to a bike.

<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<table>
<tr><td>fin</td><td>C99</td></tr>
<tr><td>

```csharp
Bike bike;
```
</td><td>

```c
Bike * bike;
```
</td></tr>
</table>
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->







<br>
<br>
<br>
<br>





# `constobj<T>`
This one feels better to me. It's clearly the object that is const and not the pointer.





### parameter: constant object
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<table>
<tr><td>fin</td><td>C99</td></tr>
<tr><td>

```csharp
void use_bike(constobj<Bike> bike)
{
    //...
}
```
</td><td>

```c
void use_bike(const Bike * bike)
{
    //...
}
```
</td></tr>
</table>
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->





### parameter: memory object
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<table>
<tr><td>fin</td><td>C99</td></tr>
<tr><td>

```csharp
void use_bike(mem<Bike> bike)
{
    //...
}
```
</td><td>

```c
void use_bike(Bike bike)
{
    // bike received by copy
}
```
</td></tr>
</table>
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->





### parameter: array of memory object
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<table>
<tr><td>fin</td><td>C99</td></tr>
<tr><td>

```csharp
void use_bike(c_array<mem<Bike>> bikes)
{
    //...
}
```
</td><td>

```c
void use_bike(Bike bikes[])
{
    //...
}
```
</td></tr>
</table>
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->








### parameter: array of constant pointers to mutable object
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<table>
<tr><td>fin</td><td>C99</td></tr>
<tr><td>

```csharp
void use_bike(constobj<c_array<Bike>> bikes)
{
    //...
}
// Nicer syntax:
void use_bike(c_const_array<Bike> bikes)
{
    //...
}
```
</td><td>

```c
void use_bike(Bike * const bike[])
{
    //...
}
```
</td></tr>
</table>
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->







### parameter: array of pointers to constant objects
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<table>
<tr><td>fin</td><td>C99</td></tr>
<tr><td>

```csharp
void use_bike(c_array<constobj<Bike>> bikes)
{
    //...
}
```
</td><td>

```c
void use_bike(const Bike * bikes[])
{
    //...
}
```
</td></tr>
</table>
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->




### parameter: array of constant pointers to constant objects

<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<table>
<tr><td>fin</td><td>C99</td></tr>
<tr><td>

```csharp
// supports read-only access to c_array
void use_bike(constobj<c_array<constobj<Bike>>> bikes)
{
    //...
}
```
</td><td>

```c
void use_bike(const Bike * const bikes[])
{
    //...
}
```
</td></tr>
</table>
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->






### parameter: constant array of pointers to constant objects
There is no such thing as a constant array in C. The contents of the array can be constant, but there's no way to declare the array variable as constant. However, array variables (that haven't decayed to a pointer) cannot be reassigned to `error: assignment to expression with array type`.

But, if the array is passed as an argument, it auto decays to a pointer, and the pointer can be reassigned. There's actually no way to declare that the array variable itself is constant (without using pointer syntax).

Examples: https://rextester.com/MWU63106

Fin can have array function parameters that are constant, but C99 cannot (without pointer syntax). That's fine though. We don't need the C array to be constant because the fin code won't transpile if the array is reassigned.

<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<table>
<tr><td>fin</td><td>C99</td></tr>
<tr><td>

```csharp
// supports read-only access to c_array
void use_bike(in c_array<constobj<Bike>> bikes)
{
    // bikes cannot be reassigned. good.
}
```
</td><td>

```c
void use_bike(const Bike * bikes[])
{
    // bikes CAN be reassigned. bad.
    // but transpile would fail, so no worries.
}
```
</td></tr>
</table>
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->






### parameter: array of constant pointers to constant objects
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<!-- ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ -->
<table>
<tr><td>fin</td><td>C99</td></tr>
<tr><td>

```csharp
void use_bike(c_const_array<constobj<Bike>> bikes)
{
    //...
}
```
</td><td>

```c
void use_bike(const Bike * const bike[])
{
    //...
}
```
</td></tr>
</table>
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->
<!-- ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ -->




