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

Signed integers are assumed to be stored using two's complement representation.

Scalar examples:
 - `u(4)` - 4-byte unsigned integer, displayed as decimal.
 - `s(8, hex, big)` - 8-byte big endian (most significant bit first) signed integer, displayed in hexadecimal.
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

## Default Command

The `default` command is used to specify global default settings. Currently the only setting available is byte order.

```C++
default(endianess = big);
// or 
default(endianess = little);
```

Byte order is `little` unless specified earlier as `big`. Note that `default` will apply to all lines of texts after it appears.

```C++
default(endianess = big);

struct Foo
{
    u(4) bigNumber;
};

default(endianess = little);

struct Bar
{
    u(4) littleNumber;
};

struct FooBar
{
    Foo foo; // foo scalars are big-endian
    Bar bar; // bar scalars are little-endian
};

layout FooBar;
```

## Layout Command

The `layout` command tells BEdit what type the binary file starts with. Multiple `layout` commands may be used, they are then interpreted to be after each other.

```C++
struct A { u(4) a; };
struct B { u(4) b; };

layout A; // Start address of A is zero.
layout B; // Start address of B is four.
```

If a layout file lacks the `layout` command no members are declared of the binary file.

## Struct Types

Structs are more like functions than C structs, you can pass parameters to it and evaluate expressions, but they serve the same purpose - to define what members a composite type contains.

The syntax is:
```
struct optional-category-specification name-of-struct optional-param-list
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

If you wish to fetch the type dynamically, you need to specify a category identifier and a type number as:
```C++
struct(CategoryIdentifier, 1234) StructName 
{
    ....
};
```

You may then use the type as part of _struct member declarations_ to do a dynamic look-up, that is you can dynamically select the type of the member based on the category specified and an integer expression.

All types of a category must have the same amount of parameters (if any).

### Statements
#### Struct Member Declarations

Members are declared by `optional-address-specifier` `type-declaration` `member-name` `optional-array-size-specifier` `;` where `type-declaration` is one of: `scalar-declaration`, `struct-declaration` (`struct-name` `optional-param-list`) or `category-declaration` (`category-identifier` `[` `integer-expression` `]` `optional-param-list`).

By default a member declared has the address of the previous member declared plus the size of that member, or zero for the first member. Observe that no padding is applied to guarantee that the members are aligned. To overwrite the address of a member, use an address specifier:
```
@(integer-expression)
```
See [Integer Expressions](#integer-expressions) for details.

Note that the address specifier makes subsequent members declared after that member, appear after it in memory. If this is not wanted you can use the `external` keyword. This can be useful if you have data that is "far away" from the struct you're declaring, but you still would like to treat it as a member.
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
In the above example, `age` has the address of `address of nameSize + 4`. If the `name` member wouldn't have been declared `external` the address would've been `nameLocation + nameSize`.

To get the current address (that is, the address the next member would be declared at) use `@` (or now deprecated `current_address()`).

Any member may be declared as `hidden` to exclude that member from being presented by BEdit. To hide a member add `hidden` before the type declaration, after address specifier if present. A hidden member can still be used in integer expressions, it only hides the value from being presented. Note that adding `hidden` before a scalar declaration is the same as declaring the scalar using `hidden` type.

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

To dynamically choose type based on some expression, declare the type using a category with a value specifying which type to use as:
```C++
struct(MyCategory, 1) Foo { ... };
struct(MyCategory, 2) Bar { ... };

struct FooBar
{
    u(4) selector;
    MyCategory[selector - 1] myMember;
};
```

If the category with the type number doesn't exist (in above case `selector - 1`) an error is reported and execution is aborted. You can use `has_type(category, number)` to query if the type exists.

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

Another way to do the same is with categories as:
```C++
enum Shape
{
    Circle,
    Rectangle,
    ...
};

struct(ShapeType, Shape.Circle) Circle
{
    f(4) x;
    f(4) y;
    f(4) radius;
};

struct(ShapeType, Shape.Rectangle) Rectangle
{
    f(4) minX;
    f(4) minY;
    f(4) width;
    f(4) height;
};

...

struct Shape(var type)
{
    if (has_type(ShapeType, type))
    {
        ShapeType[type] shape;
    }
    else
    {
        print("Unrecognized type, ignoring.");
    }
};
```

#### Loops

`for`, `while` and `do-while` loops are supported in the Layout Language, but are slightly different.

For loop is in form: `for` `(` `var` `var-name` `=` `integer-expression` `;` `integer-expression` `;` `assignment-expression` `)` `{` ... `}` (observe again the `{` and `}` are mandatory). Note that the _comma-operator_ is not supported.

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

`while` and `do-while` loops are very similar to the C variant, except that the Layout Language doesn't have a concept of scope. That means a `do-while` loop can use a variable declared inside the block in the loop-terminating expression.

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

Local variables (and struct parameters) may be mutated through assignment expressions. These are in the form `variable-name` `assignment-operator` `integer-expression`. The Layout Language doesn't support increment and decrement operators (`++i`, `i--` and similar), currently you have to write the longer expression. But the same compound assignment operators as in C are supported (`<<=`, `>>=`, `^=`, `|=`, `&=`, `%=`, `/=`, `*=`, `-=`, `+=`, `=`).

```C++
struct Foo
{
    var i; // Default value is zero.
    i += 1;
    // Or equivalent: i = i + 1;
    ...
};
```

Unlike C the assignment expression _may not_ appear inside an integer expression. That is

```C++
struct Foo
{
    var i;
    var j;
    
    i = 1 + (j += 5); // Not supported.
};
```

is not supported.

#### Integer Expressions

Integer expressions consist of _operators_ and _values_. All math in the Layout Language is done with signed 64-bit integers, as such there is no integer promotion during operators. The operators supported are similar to C.

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
% modulo

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

Note that the Layout Language lacks both unary `+` and the ternary operator `?:`. Also note that `abs` is treated as a unary operator, this means `abs -1 * -5` evaluates to `-5` (and `abs(-1 * -5)` is `5`). Operator precedence and associativity is the same as in C (`abs` has same precedence as the other unary operators).

Values can be integer literals, string literals, struct members, enum values, local variables, `@` (or deprecated `current_address()`), `size_of_file`, `has_type(category, typeNumber)`, `sizeof(...)` or sub-expressions `(...)`.

Integer literals supported are: 
- binary (prefix `0b` or `0B`)
- hexadecimal (prefix `0x` or `0X`)
- octal (prefix `0`)
- decimal

Any single-quote character (`'`) inside the literal is ignored while parsing. Unlike C++14 this character is ignored even if it's the first character after the prefix.

```
var value = 0b'0000'0000''1111'1111 == 0x0f;
```

String literals may be used as values if it can fit a 64-bit signed integer. The string is interpreted in byte order, that is:
```C++
// ASCII: 'a' = 0x64  ' ' = 0x20
"a " == 0x2064 // Note that integer literals are written with most significant digits first.
```

Escape sequences supported in string literals are `\n`, `\r`, `\\`, `\t` and `\0`.

Struct scalar members may be used as values if they can be interpreted as a 64-bit signed integer. For nested members use `.` for access as:
```C++
struct Bar
{
    u(2) value;
};

struct Foo
{
    Bar bar;
    s(8) otherValue;
    
    var a = bar.value;
    var b = otherValue + a;
};
```

You cannot access nested members of members declared as using a category declaration.

If a member appears multiple times (due to a loop expression), the most recent value is used when loading it.

Note that byte order swapping may occur during loading of members
```C++
// Expected file content: AA BB BB AA
struct Data
{
    u(2, big) a;
    u(2, little) b;
    
    assert(a == b);
};
```

To get the current address (the address the next member would be declared at) use `@` (or deprecated `current_address()`). This value changes everytime you declare a non-external member. You can also modify it by using the _address-specifier_ when declaring a member, or through a direct assignment as a separate statement. `size_of_file` can be used to get the total size of the binary file being evaluated.

```C++
struct Foo
{
    string(4) magic;
    
    var remainingSizeOfFile = size_of_file - @;
    raw(remainingSizeOfFile) remainingData;
};
```

`sizeof` is similar to as it is in C, however as the size of a type in BEdit may be dependent on the data of the file `sizeof` has an optional address specified. Syntax is `sizeof` `(` `optional-address-specifier` `struct-type` `)`. If the address specifier is not present the current address is used.

```C++
// Expected file content 01 02 .. FE FF

struct Bar
{
    u(1, hex) a;
    raw(a) b;
};

struct Foo
{
    print(sizeof(Bar));      // Prints '2'
    Bar bar;
    print(sizeof(Bar));      // Prints '4'
    print(sizeof(@(1) Bar)); // Prints '3'
    
    // print(sizeof(bar));   // Not supported.
    // print(sizeof(u(4));   // Not supported, but it's 4.
};

layout Foo;
```

#### Debugging: Print and Assert

To print a value use `print(integer-expression)`, `print("message", integer-expression)` or `print("message")`. The message and result of the integer-expression is displayed by the GUI at a suitable location (usually after the previously declared member).

To stop execution and report a fatal error use `assert(integer-expression)`.

## Example - BMP

```C++
enum PixelCompression
{
    BI_RGB,
    BI_RLE8,
    BI_RLE4,
    BI_BITFIELDS,
    BI_JPEG,
    BI_PNG,
    BI_ALPHABITFIELDS,
    BI_CMYK = 11,
    BI_CMYKRLE8,
    BI_CMYKRLE4,
};

struct FileHeader
{
    string(2) Signature;
    assert(Signature == "BM");

    u(4) FileSize;
    raw(2) Reserved[2];
    u(4, hex) FileOffsetToPixelArray;
};

struct InfoHeader
{
    u(4) Size;
    assert(Size == 40);

    s(4) Width;
    s(4) Height;

    u(2) PlaneCount;
    u(2) BitsPerPixel;
    PixelCompression(4) Compression;
    u(4) ImageSize;
    u(4) XPixelsPerMeter;
    u(4) YPixelsPerMeter;
    u(4) ColorsInColorTable;
    u(4) ImportantColorCount;
};

struct PixelRow(var compression, var bitsPerPixel, var width)
{
    assert(bitsPerPixel % 8 == 0);
    var sizeInBytes = 4 * ((bitsPerPixel * width + 31) / 32);
    var paddingInBytes = sizeInBytes - bitsPerPixel * width / 8;

    raw(bitsPerPixel / 8) Pixels[width];

    if (paddingInBytes)
    {
        raw(paddingInBytes) Padding;
    }
};

struct Bitmap
{
    FileHeader Header;
    InfoHeader Info;

    @(Header.FileOffsetToPixelArray) PixelRow(Info.Compression, Info.BitsPerPixel, Info.Width) Rows[abs Info.Height];
};

layout Bitmap;
```

## Example - PNG

```C++
default(endianess = big);

struct Header
{
	raw(1) transmissionByte;
	string(3) pngMagic;
	assert(pngMagic == "PNG");
    
	string(2) lineEnding_DOS;
	string(1) eofChar;
	string(1) lineEnding_UNIX;
};

enum sRGB_Intent
{
	Perceptual,
	RelativeColorimetric,
	Saturation,
	AbsoluteColorimetric,
};

struct(Chunk, "sRGB") sRGB(var size)
{
	assert(size == 1);
	sRGB_Intent(1) intent;
};

enum IHDR_ColorType
{
	Grayscale,
	RGB = 2,
	Palette,
	GrayscaleWithAlpha,
	RGBWithAlpha = 6
};

enum IHDR_Filter
{
	None,
	Sub,
	Up,
	Average,
	Paeth,
};

enum IHDR_Interlace
{
	None,
	Adam7,
};

struct(Chunk, "IHDR") IHDR(var size)
{
	assert(size == 13);
	
	u(4) width;
	u(4) height;
	u(1) bitDepth;
	IHDR_ColorType(1) colorType;
	u(1) compressionMethod;
	assert(compressionMethod == 0);
	
	IHDR_Filter(1) filterMethod;
	IHDR_Interlace(1) interlaceMethod;
};

struct(Chunk, "gAMA") gAMA(var size)
{
	assert(size == 4);
	u(4) factor;
};

enum pHYs_Unit
{
	Unknown,
	Meters,
};

struct(Chunk, "pHYs") pHYs(var size)
{
	assert(size == 9);
	
	u(4) ppuX;
	u(4) ppuY;
	pHYs_Unit(1) unit;
};

struct(Chunk, "tEXt") tEXt(var size)
{
	string(size) value;
};

struct(Chunk, "IDAT") IDAT(var size)
{
	hidden(size) data;
	print("Pixel data parsing left as reader's exercise.");
};

struct Chunk
{
	u(4) size;
	string(4) type;
    
    if (has_type(Chunk, type))
    {
        Chunk[type](size) chunk;
    }
    else
    {
        if (size)
        {
            raw(size) data;
        }
    }
    
	u(4) crc;
};

struct PNG
{
	Header header;
	Chunk ihdrChunk;
	assert(ihdrChunk.type == "IHDR");
	
	while (@ < size_of_file)
	{
		Chunk chunk;
	}
};

layout PNG;
```
