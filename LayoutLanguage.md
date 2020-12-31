# Layout Language

The layout language is the programming language you use to declare what's in the binary file. The language is loosely based on C and several keywords are borrowed from there.

In general, the layout definition file contains a list of type declarations followed by one or more layout instructions.

```C++
struct MyData
{
    raw(64) first_64_bytes_in_file;
};

layout MyData;
```

The example above will display the first 64 bytes in the file as uninterpreted bytes. If the file contains less than 64 bytes worth of data in it, an error is presented.

## Comments

Both `//` and `/* ... */` style comments are supported. The only difference between C and the Layout Language is that `/* ... */` comments may be nested.

## Scalar Types

Unlike C, the layout language doesn't have *int*, *short* or *long*. Instead the scalar types are defined by type, radix and byte order.

Syntax is: `type ( size-in-bytes, optional-radix, optional-byte-order )`

Types are:
 - `raw` - bytes are displayed as hexadecimal values, in memory order.
 - `u` - unsigned integer.
 - `s` - signed integer.
 - `f` - floating point.
 - `string` - ASCII string.
 - `hidden` - bytes are not displayed.
 - `<enum-name>` - bytes are displayed as an enum member or bitwise flags thereof.

Byte order, when present, will overwrite the default byte order for that scalar. Default byte order (unless specified otherwise with the default statement) is little endian (least significant byte first).

Radixes are: `binary` (base 2), `octal` (base 8) and `hex` (base 16). If radix specifier is not present the default base 10 is used to represent the data. Note that types string and raw will ignore the radix.

Scalar examples:
 - `u(4)` - 4-byte integer, displayed as decimal.
 - `s(8, hex, big)` - 8-byte big endian (most significant bit first) integer, displayed in hexadecimal.
 - `string(1)` - 1-byte ASCII string (similar to char in C).
 - `hidden(256)` - 256 ignored bytes.
 - `Foo(2)` - (where Foo is an enum) 2-byte enum value displayed as Foo member(s).

The size parameter does not need to be a constant. See [Variables](#variables) section for more details.

## Aliases

`typedef` is supported by the Layout Language for *scalar types*. These must be specified at global scope.
```C++
typedef u(4, hex) Hex_U32;

struct MyFile
{
    Hex_U32 signature;
    ...
};
```

## Enum Types

The Layout Language has enum types similar to C.
                                                
There are three versions of enum declarations:
```C++
// Named enum.
enum Foo_1
{
    Foo_1A, Foo_1B, Foo_1C = 5
};

// Anonymous enum.
enum
{
    A, B, C,
};

// Named anonymous enum.
enum anonymous Foo_2
{
    Foo2_A, Foo2_B = 8, Foo2_C
};
```
See [Scalar types](#scalar-types) for how to define an enum member.

All three versions of enum declarations are identical with the exception of how you access the enum value. For anonymous enums (both named an unnamed) you can access the enum value by name. For non-anonymous enums you must prefix them like 'name-of-type.name-of-value'. That is:
```C++
x = Foo_1.Foo_1B; // Value of first enum's (Foo_1) Foo_1B
x = A;            // Value of second enum's A
x = Foo2_A;       // Value of third enum's (Foo_2) Foo_A
x = Foo2.Foo2_A;  // Same as previous line
```
Just like in C the first enum value is implicitly zero and any other enum value is the previous value plus one. You can also assign values using an assignment operator '='.

Note that unlike C, the enum doesn't have an implicit size. The size of the type is specified in the scalar declaration.

## Struct Types

Structs are more like functions than C structs, you can pass parameters to it and evaluate expressions, but they serve the same purpose - to define what members a composite type contains.

The syntax is:
```
struct name-of-struct optional-param-list
{
    statements...
};
```
As an example, we can use the following struct to declare a bmp-file header:
```C++
struct BMPHeader
{
    string(2) Signature;
    assert(Signature == "BM");

    u(4) FileSize;
    hidden(2) Reserved[2];
    u(4, hex) OffsetToPixelData;
};
```
For all available statements see [Statements](#statements) section.

Note that there is no padding added between the members, so take extra care when copy pasting C-struct definitions.

To specify that a struct takes parameters add `(var a, var b, ..., var n)` after the struct name as:
```C++
struct StructWithParams(var paramA, var paramB, var paramC)
{
    ...
};
```

All variables are interpreted as 64-bit signed integers.

### Statements
#### Struct Member Declarations

Members are declared by `optional-address-specifier` `scalar-type-declaration` or `struct-name` `optional-array-size-specifier` `;`.

By default a member declared has the address of the previous member declared plus the size of the member, or zero if it's the first member. Observe that no padding is applied to guarantee that the members are aligned. To overwrite the address of a member, use an address specifier:
```
@(integer-expression)
```
See [Integer Expression](#integer-expression) for details.

Note that the address specifier makes subsequent members declared after that member, appear after it. If this is not wanted you can use the `external` keyword. This can be useful if you have data that is "far away" to the struct you're declaring, but you still would like to treat it as a member.
```C++
struct UserData
{
    u(8, hex) nameLocation;
    u(4) nameSize;
    @(external nameLocation) string(nameSize) name;
    u(4) age;
    ...
};
```
In the above example, `age` has the address of `address-of-nameSize + 4`. If the `name` member wouldn't have been declared `external` the address would've been `nameLocation + nameSize`.

To get the current address (that is, the address the next member would be declared at) use `current_address()`. See [In-built Operators](#in-built-operators) for details.

When specifying a scalar member see [Scalar types](#scalar-types), for a member that is a struct use the name of the type and any parameter it takes (if any) in parenthesis as:
```C++
struct Foo(var a) { ... };
struct Baz { ... };
struct Bar
{
    Foo(8) foo;
    Baz baz;
};
```

Declaring an array is similar to how you would do it in C (with `[array_size]`), but the size of the array may be a result of an [integer expression](#integer-expression).

```C++
struct Bar
{
    u(4) len;
    raw(1) data[len];
};
```

Note that the Layout Language doesn't allow multiple members on same line (`int x, y;`), each member must be a separate statement.

#### Variables

Inside a struct declaration you may declare local variables. The syntax is `var` `variable-name` with optional `=` `integer-expression` followed by `;`. If the optional assignment part is missing the variable is initialized to zero. All local variables are 64-bit signed integers.

```C++
struct Foo
{
    var eight = 8;
    var nine = eight + 1;
    u(eight) data;
};
```

Struct parameters are treated as local variables inside the struct declaration. The Layout Language doesn't have a concept of scopes other than struct declarations, this means you can access local variables declared inside a "block" on the "outside".

#### If Statements

`if`, `else if` and `else` are very similar to C and many other languages. One of the main differences between C and the Layout Language is that `{` and `}` are mandatory. If-statements also provides a way to simulate `unions` in the Layout Language:
```C++
enum Shape
{
    Circle,
    Rectangle,
    ...
};

struct Circle
{
    f(4) x;
    f(4) y;
    f(4) radius;
};

struct Rectangle
{
    f(4) minX;
    f(4) minY;
    f(4) width;
    f(4) height;
};

...

struct Shape(var type)
{
    if (type == Shape.Circle)
    {
        Circle circle;
        // NOTE circle size is 4 * 3, the size of this type is 12 bytes when hitting this branch.
    }
    else if (type == Shape.Rectangle)
    {
        Rectangle rectangle;
        // NOTE rectangle size is 4 * 4, the size of this type is 16 bytes when hitting this branch.
    }
    else if (...)
    {
        ...
    }
};
```

#### Loops

`for`, `while` and `do-while` loops are supported in the Layout Language, but are slightly different.

For loop is in form: `for` `(` `var` `var-name` `=` `integer-expression` `;` `integer-expression` `;` `assignment-expression` `)` `{` ... `}` (observe again the `{` and `}` are mandatory).

```C++
struct Foo
{
    u(4) count;
    for (var i = 0; i < count; i = i + 1)
    {
        raw(1) data;
    }
};
```

`while` and `do-while` loops are very similar to the C variant, except that the Layout Language doesn't have a concept of scope. That means a `do-while` loop can use a variable declared inside the scope in the loop-terminating expression.

```C++
struct Foo
{
    do
    {
        u(4) data;
        var stop = (data != 16);
    }
    while (!stop);
};
```

#### Assignment Expressions

Local variables (and struct paramters) may be mutated through assignment expressions. These are in the form `variable-name` `=` `integer-expression`. The Layout Language doens't support `++i` or similar, currently you have to write the longer expression.

```C++
struct Foo
{
    var i; // Default value is zero.
    i = i + 1;
    ...
};
```

#### Integer Expressions

All math in the Layout Language is done with signed 64-bit integers. The operators supported are similar to C.

Unary operators are:
```
! logical not
~ bitwise not
- negation
abs - absolute value
```

Binary operators are:
```
* multiplication
/ division
+ addition
- subtraction

< less than
<= less than or equal
> greater than
>= greater than or equal
== equal
!= not equal
|| logical or
&& logical and

<< bitshift left
>> bitshift right
& bitwise and
| bitwise or
^ bitwise xor
```

Note that the Layout Language lacks both unary `+` and the ternary operator `?:`. Also note that `abs` is treated as a unary operator, this means `abs -1` evaluates to `1`. Operator precedence and associativity is the same as in C (`abs` has same precedence as the other unary operators).

Parenthesis may be added to change precedence, `2 * 5 + 1` evaluates to `11` while `2 * (5 + 1)` evaluates to `12`.
