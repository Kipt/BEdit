This document was written for BEdit version 0.2.2 and includes some changes not available in earlier versions. See [GitHub](https://github.com/Kipt/BEdit) for latest version.

# Layout Language

This document introduces all current features by going through some use-cases. Referenced files can be found in BEdit package (both the free open source one, and paid version) as well as in the [public repository](https://bitbucket.org/Kipt88/bedit_pub/src/master/examples/). The layout language features are not tied to the paid version.

As many features have similar C counter parts, a familiarity with C is expected from the reader.

## Introduction

The layout language is the programming language you use to declare the format of a binary file. The language is loosely based on C and several keywords and concepts are borrowed from there.

To follow the examples here you can use the command line or GUI version of BEdit. For command line, invoke it as 

```bash
bedit -data path/to/your/binary/file -layout path/to/your/source/file
```

and for the GUI version you can either use the main menu drop down menus or simply drag'n'drop both the binary file and source file on the window. BEdit GUI assumes the source file has `.bet` as file extension when dropping on window.

A typical layout definition file contains a list of type declarations followed by one or more layout commands. As a simple example, the below specifies the format of the binary file to start with 10 bytes that are displayed as "raw":

```C++
struct MyData
{
    // This assumes the file contains at least 10 bytes.
    raw(10) first_10_bytes_in_file;
};

layout MyData;
```

If there are more than 10 bytes in the file the extra bytes are ignored, and if there are less an error is presented.

`struct MyData { ... }` should look familiar to you if you have programmed in C, this is a composite type definition named `MyData`. A `struct` contains a list of members specified with a type followed by member name. In above case it only has one member, `first_10_bytes_in_file` of type `raw(10)`.

The type of a member tells BEdit two things, how to display it and how big it is. `raw(10)` specifies that it should be displayed as "raw" and the size (in bytes) is 10.

You can create any amount of `struct` types in the source file, so we need to tell BEdit which type the file starts with. This is done with the `layout` command. If you specify multiple `layout` commands they are interpreted in order.

```C++
struct MyData
{
    raw(10) ten_bytes;
};

layout MyData; // Displays first 10 bytes
layout MyData; // Displays next 10 bytes
```

All data from the binary file is read from the current address. This starts at zero and advances every time a member is evaluated. The first layout command displays `raw(10)` at address zero and advances current address by 10, the second layout command displays `raw(10)` at address `10` (that is `0x0A`).

## Scalar Types

All members of a `struct` have a type, this is either a scalar, `enum`, or another `struct` type previously declared.

Scalar types are defined by base type, size, radix and byte order in the form:
```C++
base-type(size, radix, byte-order)
```

Base type of a scalar determines how the data is loaded and displayed. Valid base types are:

- `raw` - data loaded and displayed in memory order,
- `u` - unsigned integer,
- `s` - signed integer (using two's complement),
- `f` - floating point,
- `string` - ASCII string and
- `hidden` - data not displayed.

The radix of a scalar type determines what numerical base should be used. These are:

- `binary` - base 2,
- `octal` - base 8,
- `hex` - base 16.

By default the radix is 10, except for base type `raw` that uses hexadecimal. `string` and `hidden` does not have a radix specifier. Currently `f` ignores radix specifier and is always displayed in decimal form.

Byte order is by default `little` (unless overwritten by the `default` statement). To explicitly specify byte order `little` or `big` can be added to the scalar type. Note that byte order cannot be specified for `raw` or `string` as they are always loaded and displayed in memory order.

Here are some common examples:

- `u(4, hex)` - 32-bit unsigned integer, displayed in hexadecimal
- `s(8, big)` - 64-bit big endian signed integer, in decimal
- `f(4)` - single precision float
- `raw(64)` - 64 bytes of raw data, displayed in hexadecimal
- `hidden(256)` - 256 bytes that are not displayed

Note the difference between `raw(size)` and `u(size, hex)`, `u` might perform a byte order swap. To indicate the difference between these two types, `raw` numbers are displayed in BEdit with a prefix (`0b`, `0`, `0x`), all others with a suffix (`b`, `o`, `h`).

Let's consider a [Bitmap File Header](https://en.wikipedia.org/wiki/BMP_file_format#Bitmap_file_header). It has five members starting with 2 byte file magic, 4 byte unsigned integer telling size of file, two 2-byte members that are reserved and lastly 4 byte unsigned integer offset to start of pixels.

```C++
struct BMPFileHeader
{
    string(2) signature; // Expect to be "BM"

    u(4) fileSize;
    raw(2) reserved1;
    raw(2) reserved2;
    u(4, hex) fileOffsetToPixelArray;
};

layout BMPFileHeader;
```

If you run the above using `examples/test.bmp` that's inside the BEdit package you will see BEdit layout the content as

```
BMPFileHeader
 0h              signature   "BM"
02h               fileSize    306
06h              reserved1 0x0000
08h              reserved2 0x0000
0Ah fileOffsetToPixelArray    36h
```

As displaying `reserved1` and `reserved2` is of little value we could also change those to

```C++
    hidden(2) reserved1;
    hidden(2) reserved2;
```

or using an array

```C++
    hidden(2) reserved[2];
```

## Typedef

In above header I chose to display the offset as a hexadecimal value `u(4, hex) fileOffsetToPixelArray`. This is a personal preference as it could just as well be written in decimal (or even octal!). To make the file offset number format easier to change you can add a type alias using `typedef`.

```C++
typedef u(4, hex) offset_t;

struct BMPFileHeader
{
    string(2) signature;
    u(4) fileSize;
    hidden(2) reserved[2];

    offset_t fileOffsetToPixelArray;
};

layout BMPFileHeader;
```

Note that `typedef` can only be applied to scalar types, not `enum` or `struct`.

## Asserts and Prints

In the above BMP file header example we have a file signature. This is very common in binary files and is used to detect if the file is in the expected format.

We can extend our previous example to check that the file is indeed having `BM` as signature by adding an `assert`.

```C++
typedef u(4, hex) offset_t;

struct BMPFileHeader
{
    string(2) signature;
    assert(signature == "BM");
    
    u(4) fileSize;
    hidden(2) reserved[2];
    offset_t fileOffsetToPixelArray;
};

layout BMPFileHeader;
```

`assert` takes a single parameter that if evaluates to false, stops the execution immediately.

The output remains the same as the `assert` is not triggered, but now let's try to open an invalid file, such as `examples/test.png`.

```
BMPFileHeader
0h signature "\x89P"
doc_bmp.bet(5,5): assert(signature == "BM");
                  ~~~~~~
Assertion triggered!
```

The assertion stops evaluation of the members immediately and shows what file and where the issue was.

For displaying messages that shouldn't stop execution you can use `print`. Let's print out if reserved is zero or not

```C++
typedef u(4, hex) offset_t;

struct BMPFileHeader
{
    string(2) signature;
    assert(signature == "BM");
    
    u(4) fileSize;
    hidden(2) reserved[2];

    if (reserved[0] == 0 && reserved[1] == 0)
    {
        print("reserved are both zero");
    }
    else
    {
        print("reserved[0] = ", reserved[0]);
        print("reserved[1] = ", reserved[1]);
    }

    offset_t fileOffsetToPixelArray;
};

layout BMPFileHeader;
```

`print` takes one or two parameters, the message and an optional expression as second parameter. You can also omit the message and just pass the expression, then the message is the text that makes the expression (e.g. `print(1 + 3)` will display "`(1 + 3) 4`").

## Literals

We used `signature == "BM"` in above example, something that you can't do in C.

All expressions you write in BEdit are using signed 64-bit integer math. `signature` is loaded as an integer and then compared with `"BM"` that has been loaded as well. This means that string literals that fit in a 64-bit integer can be used as numbers. Note that attempting to use a member, or string literal, larger than 64-bit in an expression is an error.

String literals can be quoted with either single (`'`) or double quote (`"`) marks and characters are escaped similar as you would in C.

Integer literals are same as in C with the exception that you can add `'` wherever you like in the literal after any prefix, it is ignored while parsing (similar to C++14) like `0xAA'BB'12` or `0b'1111'0000''1100'0000`.

Note that the prefix can cause some confusion. The literal `0xAABB` means `0xAA * 0x100 + 0xBB`, that is `0xAA` is the most significant byte and `0xBB` is the least, but `raw` that displays with a prefix is in memory order. On little-endian layouts (the default) this means memory printed by BEdit as `0xAABB` equals the literal `0xBBAA`.

## Enum

To continue on with the BMP example, after the file header there is an info header. Adding the info header and adding a new layout is fairly straight forward following the Wikipedia article.

```C++

typedef u(4, hex) offset_t;

struct BMPFileHeader
{
    ...
};

struct BMPInfoHeader
{
    u(4) size;
    
    s(4) width;
    s(4) height;
    
    u(2) planeCount;
    u(2) bitsPerPixel;
    
    u(4) compression; // 0 = BI_RGB, 1 = BI_RLE8, ...
    
    u(4) imageSize;
    u(4) xPixelsPerMeter;
    u(4) yPixelsPerMeter;
    u(4) colorsInColorTable;
    u(4) importantColorCount;
};

layout BMPFileHeader;
layout BMPInfoHeader;
```

Running that with `examples/test.bmp` shows the additional type as:

```
BMPInfoHeader
0Eh                   size   40
12h                  width   11
16h                 height    7
1Ah             planeCount    1
1Ch           bitsPerPixel   24
1Eh            compression    0
22h              imageSize  252
26h        xPixelsPerMeter    0
2Ah        yPixelsPerMeter    0
2Eh     colorsInColorTable    0
32h    importantColorCount    0
```

We can see that the `compression` is zero, that corresponds to `BI_RGB`. To tell BEdit to format it as such we create an `enum`.

```C++
typedef u(4, hex) offset_t;

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

struct BMPFileHeader
{
    ...
};

struct BMPInfoHeader
{
    ...

    PixelCompression(4) compression;

    ...
};

layout BMPFileHeader;
layout BMPInfoHeader;
```

Adding that `enum` as type to `compression` prints out

```
1Eh            compression BI_RGB
```

instead of the number. This also works for flag-like enums, "bitwise or":s will be added to indicate what masks are set.

The declaration of an `enum` member (`PixelCompression(4) compression;`) is very similar to that of a scalar but the `enum` only has a size parameter. The size can be omitted if the `enum` was declared as `enum PixelCompression(4) { ... };` instead.

Accessing `enum` members are slightly different from C. To use `BI_RGB` in an expression you write `PixelCompression.BI_RGB` instead of just `BI_RGB`. If this is unwanted behavior you can declare the `enum` type as `anonymous`.

```C++
enum anonymous PixelCompression(4)
{
    BI_RGB,
    ...
};
...
struct BMPInfoHeader
{
    ...
    PixelCompression compression;
    assert(compression == BI_RGB);
    ...
};
```

## Member Access and Current Address

To display the pixels we need to access `BMPFileHeader.fileOffsetToPixelArray` and other members like `bitsPerPixel`. This cannot be done after the `layout` has finished, as such we need to put the headers in one `struct`.

```C++

enum anonymous PixelCompression(4)
{
    ...
};

struct BMPFileHeader
{
    ...
};

struct BMPInfoHeader
{
    ...
};

struct BMPFile
{
    BMPFileHeader fileHeader;
    BMPInfoHeader infoHeader;
};

layout BMPFile;
```

Now we can use the nested members to display the first row.

```C++
struct BMPFile
{
    BMPFileHeader fileHeader;
    BMPInfoHeader infoHeader;

    assert(infoHeader.bitsPerPixel % 8 == 0);

    raw(infoHeader.bitsPerPixel / 8) firstRow[infoHeader.width];

    // There might need to be padding before next row.
};
```

Each pixel is displayed as `raw`, and there is `infoHeader.width` amount of them.

For the `examples/test.bmp` the above works, but just because of luck. The address of `firstRow` must be `fileHeader.fileOffsetToPixelArray`.

To specify the address of a member we can use an address specifier `@(address)` before creating the member.

```C++

struct BMPFile
{
    BMPFileHeader fileHeader;
    BMPInfoHeader infoHeader;
    
    assert(infoHeader.bitsPerPixel % 8 == 0);
    
    @(fileHeader.fileOffsetToPixelArray)
        raw(infoHeader.bitsPerPixel / 8) firstRow[infoHeader.width];
    
    // There might need to be padding before next row.
};
```

`@` can also be used on its own as a global variable. This is the absolute address in the file where the next member is going to be defined. So alternatively we could assign it directly

```C++
    @ = fileHeader.fileOffsetToPixelArray;
    raw(infoHeader.bitsPerPixel / 8) firstRow[infoHeader.width];
```

## Local Variables and Type Parameters

The size of a pixel row of a BMP image in bytes (including padding) can be calculated as `4 * floor((bitsPerPixel * width + 31) / 32)`. To make a pixel row its own `struct` it would look something like:

```C++
struct PixelRow
{
    raw(bitsPerPixel / 8) pixels[width];
    if (4 * ((bitsPerPixel * width + 31) / 32) - bitsPerPixel * width / 8 > 0)
    {
        raw(4 * ((bitsPerPixel * width + 31) / 32) - bitsPerPixel * width / 8) padding;
    }
};
```

which has several issues. First, we don't have access to the `bitsPerPixel` and `width` members and second the repeating of calculations makes the `struct` hard to read.

`struct` types in BEdit can take parameters. In fact, a `struct` is more like a function than a type. All parameters are 64-bit signed integers.

```C++
...

struct PixelRow(var bitsPerPixel, var width)
{
    assert(bitsPerPixel % 8 == 0);
    
    raw(bitsPerPixel / 8) pixels[width];
    
    // There might need to be padding before next row.
};

struct BMPFile
{
    BMPFileHeader fileHeader;
    BMPInfoHeader infoHeader;
    
    @(fileHeader.fileOffsetToPixelArray) PixelRow(infoHeader.bitsPerPixel, infoHeader.width) firstRow;
};
```

`var` is not only used to specify `struct` parameters, it is also used to create local variables. We can add the padding as:

```C++
...

struct PixelRow(var bitsPerPixel, var width)
{
    assert(bitsPerPixel % 8 == 0);
    
    var sizeInBytes = 4 * ((bitsPerPixel * width + 31) / 32);
    var paddingInBytes = sizeInBytes - bitsPerPixel * width / 8;
    
    raw(bitsPerPixel / 8) pixels[width];
    
    if (paddingInBytes)
    {
        raw(paddingInBytes) padding;
    }
};

...
```

If a variable is not assigned a value at definition, it defaults to a value of zero.

The resulting display looks like:

```
PixelRow (.firstRow)
36h             pixels[ 0] 0xFF00FF
39h             pixels[ 1] 0xFFFFFF
3Ch             pixels[ 2] 0xFFFFFF
3Fh             pixels[ 3] 0xFFFFFF
42h             pixels[ 4] 0xFFFFFF
45h             pixels[ 5] 0xFFFFFF
48h             pixels[ 6] 0xFFFFFF
4Bh             pixels[ 7] 0xFFFFFF
4Eh             pixels[ 8] 0xFFFFFF
51h             pixels[ 9] 0xFFFFFF
54h             pixels[10] 0xFFFFFF
57h                padding 0x000000
```

## Operators

This only shows the first row of the image, to show all rows we can use `infoHeader.height` and define the rows as an array of `PixelRow`. The problem is that `infoHeader.height` might be negative. We could do:

```C++
...

struct BMPFile
{
    BMPFileHeader fileHeader;
    BMPInfoHeader infoHeader;
    
    var height = infoHeader.height;
    if (height < 0)
    {
        height = -height;
    }
    
    @(fileHeader.fileOffsetToPixelArray) PixelRow(infoHeader.bitsPerPixel, infoHeader.width) rows[height];
};

layout BMPFile;
```

but that is quite verbose.

BEdit supports most unary and binary operators of C, but the ternary operator is not supported. BEdit also adds one more operator, `abs`. We can simplify the above to:

```C++
...

struct BMPFile
{
    BMPFileHeader fileHeader;
    BMPInfoHeader infoHeader;
    
    @(fileHeader.fileOffsetToPixelArray) PixelRow(infoHeader.bitsPerPixel, infoHeader.width) rows[abs infoHeader.height];
};

layout BMPFile;
```

This should display the entire file.

### bmp.bet

```C++
typedef u(4, hex) offset_t;

enum anonymous PixelCompression(4)
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

struct BMPFileHeader
{
    string(2) signature;
    assert(signature == 'BM');
    
    u(4) fileSize;
    hidden(2) reserved[2];
    
    offset_t fileOffsetToPixelArray;
};

struct BMPInfoHeader
{
    u(4) size;
    
    s(4) width;
    s(4) height;
    
    u(2) planeCount;
    u(2) bitsPerPixel;
    
    PixelCompression compression;
    assert(compression == BI_RGB);
    
    u(4) imageSize;
    u(4) xPixelsPerMeter;
    u(4) yPixelsPerMeter;
    u(4) colorsInColorTable;
    u(4) importantColorCount;
};

struct PixelRow(var bitsPerPixel, var width)
{
    assert(bitsPerPixel % 8 == 0);
    
    var sizeInBytes = 4 * ((bitsPerPixel * width + 31) / 32);
    var paddingInBytes = sizeInBytes - bitsPerPixel * width / 8;
    
    raw(bitsPerPixel / 8) pixels[width];
    
    if (paddingInBytes)
    {
        raw(paddingInBytes) padding;
    }
};

struct BMPFile
{
    BMPFileHeader fileHeader;
    BMPInfoHeader infoHeader;
    
    @(fileHeader.fileOffsetToPixelArray) PixelRow(infoHeader.bitsPerPixel, infoHeader.width) rows[abs infoHeader.height];
};

layout BMPFile;
```

## `default` Statement

PNG files are quite different from BMP although they do represent images. The file specification can be found from [libpng.org](http://www.libpng.org/pub/png/spec/1.2/PNG-Contents.html).

PNG scalars are stored in big endian. You can write all scalar members with `big` in the type declaration but as the entire file is in same byte order, we can use the `default` statement to set byte order.

```C++
default(endianess = big);

struct PNGHeader
{
	raw(1) transmissionByte;
	string(3) pngMagic;
	assert(pngMagic == "PNG");
    
	string(2) lineEnding_DOS;
	string(1) eofChar;
	string(1) lineEnding_UNIX;
};

layout PNGHeader;
```

The `default` statement effects all lines after it, until another default statement changes the behavior.

Note that the members of the header are only `raw` and `string`, both of which ignore byte order so the effect of `default(endianess = big)` is not yet visible.

If you run the above using `examples/test.png` that's inside the BEdit package you will see BEdit layout the content as

```
PNGHeader
 0h transmissionByte   0x89
01h         pngMagic  "PNG"
04h   lineEnding_DOS "\r\n"
06h          eofChar "\x1a"
07h  lineEnding_UNIX   "\n"
```

After the header a PNG file contains chunks until end of file.

## Loops and `size_of_file`

A chunk contains a size and type identifier followed by the content and a crc. Where the first chunk type is expected to be `IHDR`.

```C++

default(endianess = big);

struct PNGHeader
{
    ...
};

struct Chunk
{
    u(4) size;
    string(4) type;
    
    if (size)
    {
        raw(size) data;
    }
    
    u(4) crc;
};

struct PNGFile
{
    PNGHeader header;
    Chunk ihdrChunk;
    assert(ihdrChunk.type == "IHDR");

    Chunk chunk1;
    Chunk chunk2;
    Chunk chunk3;
    // ...
};

layout PNGFile;
```

The chunks are displayed as 

```
Chunk (.ihdrChunk)
08h             size                           13
0Ch             type                       "IHDR"
10h             data 0x0000000B000000070806000000
1Dh              crc                3 731 797 853

Chunk (.chunk1)
21h             size                            1
25h             type                       "sRGB"
29h             data                         0x00
2Ah              crc                2 932 743 401

Chunk (.chunk2)
2Eh             size                            4
32h             type                       "gAMA"
36h             data                   0x0000B18F
3Ah              crc                  201 089 285

Chunk (.chunk3)
3Eh             size                            9
42h             type                       "pHYs"
46h             data         0x00000EC300000EC301
4Fh              crc                3 345 983 588
```

We can't create the chunks as an array because we don't know how many they are based on the header. Since they have dynamic size we can't calculate it either.

For these situations we can create a loop that declares a chunk, until end of file.

```C++
...

struct PNGFile
{
    PNGHeader header;
    Chunk ihdrChunk;
    assert(ihdrChunk.type == "IHDR");
    
    while (@ < size_of_file)
    {
        Chunk chunk;
    }
};

```

`size_of_file` is a constant containing the size of the binary file, in bytes.

```
Chunk (.ihdrChunk)
08h             size                                                       13
0Ch             type                                                   "IHDR"
10h             data                             0x0000000B000000070806000000
1Dh              crc                                            3 731 797 853

Chunk (.chunk)
21h             size                                                        1
25h             type                                                   "sRGB"
29h             data                                                     0x00
2Ah              crc                                            2 932 743 401

Chunk (.chunk)
2Eh             size                                                        4
32h             type                                                   "gAMA"
36h             data                                               0x0000B18F
3Ah              crc                                              201 089 285

Chunk (.chunk)
3Eh             size                                                        9
42h             type                                                   "pHYs"
46h             data                                     0x00000EC300000EC301
4Fh              crc                                            3 345 983 588

Chunk (.chunk)
53h             size                                                       25
57h             type                                                   "tEXt"
5Bh             data     0x536F667477617265007061696E742E6E657420342E302E3231
74h              crc                                            4 045 433 237

Chunk (.chunk)
78h             size                                                       27
7Ch             type                                                   "IDAT"
80h             data 0x285363F8CF00464400A03A288B2830341513ADFCFF7F00BB7E2CE2
9Bh              crc                                            2 976 452 423

Chunk (.chunk)
9Fh             size                                                        0
A3h             type                                                   "IEND"
A7h              crc                                            2 923 585 666
```

As the last chunk has `IEND` as type we could also write the loop as:

```C++
    do
    {
        Chunk chunk;
    }
    while (chunk.type != "IEND");
```

## `if` and `else if`

The chunk `data` is in a format decided by `type` and `size` members. Not all chunk types are guaranteed to be in the file, but at least `IHDR`, `IDAT` and `IEND` is. We can add this inside the current chunk type by branching.

The `examples/test.png` has chunks, `IHDR`, `sRGB`, `gAMA`, `pHYs`, `tEXt`, `IDAT` and `IEND`. Adding is fairly straight forward, look at the documentation for the chunk and add an `else if` branch for the type.

```C++
...

enum sRGB_Intent
{
	Perceptual,
	RelativeColorimetric,
	Saturation,
	AbsoluteColorimetric,
};

struct sRGB(var size)
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

struct IHDR(var size)
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

struct gAMA(var size)
{
	assert(size == 4);
	u(4) factor;
};

enum pHYs_Unit
{
	Unknown,
	Meters,
};

struct pHYs(var size)
{
	assert(size == 9);
	
	u(4) ppuX;
	u(4) ppuY;
	pHYs_Unit(1) unit;
};

struct tEXt(var size)
{
	string(size) value;
};

struct IDAT(var size)
{
	hidden(size) data;
	print("Pixel data parsing left as reader's exercise.");
};

struct Chunk
{
    u(4) size;
    string(4) type;
    
    if (type == "IHDR")
    {
        IHDR(size) data;
    }
    else if (type == "gAMA")
    {
        gAMA(size) data;
    }
    else if (type == "pHYs")
    {
        pHYs(size) data;
    }
    else if (type == "tEXt")
    {
        tEXt(size) data;
    }
    else if (type == "IDAT")
    {
        IDAT(size) data;
    }
    else if (type == "sRGB")
    {
        sRGB(size) data;
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

...
```

The chunks now display more useful information.

```
Chunk (.ihdrChunk)
08h              size                           13
0Ch              type                       "IHDR"
IHDR (.data)
10h             width                           11
14h            height                            7
18h          bitDepth                            8
19h         colorType                 RGBWithAlpha
1Ah compressionMethod                            0
1Bh      filterMethod                         None
1Ch   interlaceMethod                         None

1Dh               crc                3 731 797 853

Chunk (.chunk)
21h              size                            1
25h              type                       "sRGB"
sRGB (.data)
29h            intent                   Perceptual

2Ah               crc                2 932 743 401

Chunk (.chunk)
2Eh              size                            4
32h              type                       "gAMA"
gAMA (.data)
36h            factor                       45 455

3Ah               crc                  201 089 285

Chunk (.chunk)
3Eh              size                            9
42h              type                       "pHYs"
pHYs (.data)
46h              ppuX                        3 779
4Ah              ppuY                        3 779
4Eh              unit                       Meters

4Fh               crc                3 345 983 588

Chunk (.chunk)
53h              size                           25
57h              type                       "tEXt"
tEXt (.data)
5Bh             value "Software\0paint.net 4.0.21"

74h               crc                4 045 433 237

Chunk (.chunk)
78h              size                           27
7Ch              type                       "IDAT"
IDAT (.data)
"Pixel data parsing left as reader's exercise."

9Bh               crc                2 976 452 423

Chunk (.chunk)
9Fh              size                            0
A3h              type                       "IEND"
A7h               crc                2 923 585 666
```

## Categories / Reflection

The `if`, `else if`, `else`, although it does the trick is very sub-optimal. A better way to structure the `Chunk` is by using categories.

A category is a "tag" you can assign to a type with a unique type id. You can then declare a member using a type id loaded dynamically. You assign a tag and type id as

```C++
struct(TagId, typeNumber) MyType { ... };
```

and declare a member as

```C++
struct SomeType
{
   TagId[typeNumber] data;
};
```

If the `TagId` does not have `typeNumber` then an error is reported. To check for this before hand use `has_tag(TagId, typeNumber)`.

```C++
...

struct(ChunkData, "sRGB") sRGB(var size)
{
    ...
};

struct(ChunkData, "IHDR") IHDR(var size)
{
    ...
};

struct(ChunkData, "gAMA") gAMA(var size)
{
    ...
};

struct(ChunkData, "pHYs") pHYs(var size)
{
    ...
};

struct(ChunkData, "tEXt") tEXt(var size)
{
    ...
};

struct(ChunkData, "IDAT") IDAT(var size)
{
    ...
};

struct Chunk
{
    u(4) size;
    string(4) type;
    
    if (has_type(ChunkData, type))
    {
        ChunkData[type](size) data;
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
...
```

Remember that a string literal can be used as a number, if it fits in 64-bits (8 characters).

### png.bet

```C++
default(endianess = big);

struct PNGHeader
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

struct(ChunkData, "sRGB") sRGB(var size)
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

struct(ChunkData, "IHDR") IHDR(var size)
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

struct(ChunkData, "gAMA") gAMA(var size)
{
	assert(size == 4);
	u(4) factor;
};

enum pHYs_Unit
{
	Unknown,
	Meters,
};

struct(ChunkData, "pHYs") pHYs(var size)
{
	assert(size == 9);
	
	u(4) ppuX;
	u(4) ppuY;
	pHYs_Unit(1) unit;
};

struct(ChunkData, "tEXt") tEXt(var size)
{
	string(size) value;
};

struct(ChunkData, "IDAT") IDAT(var size)
{
	hidden(size) data;
	print("Pixel data parsing left as reader's exercise.");
};

struct Chunk
{
	u(4) size;
	string(4) type;
    
    if (has_type(ChunkData, type))
    {
        ChunkData[type](size) data;
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

struct PNGFile
{
    PNGHeader header;
    Chunk ihdrChunk;
    assert(ihdrChunk.type == "IHDR");
    
    do
    {
        Chunk chunk;
    }
    while (chunk.type != "IEND");
};

layout PNGFile;
```

## Converting C to BEdit

Although they look similar, C `struct` is not fully compatible with BEdit. Features that BEdit does not have are bitfields, `union`, C types, nested anonymous structs and default alignment.

You can simulate a `union` in BEdit by manually offsetting `@`. Consider:

```C++
// C++
struct MyValue
{
    bool isInt;
    union
    {
        int intValue;
        float floatValues[4];
    };
    // Size of union is 4*4 = 16 bytes.
};
```

In BEdit you could write (assuming `bool` is one byte, `int` and `float` 4 bytes).

```C++
enum anonymous bool(1)
{
    false, true
};

typedef s(4) int;
typedef f(4) float;

struct MyValues
{
    bool isInt;
    hidden(3) padding; // Assumed padding, C struct padding is compiler specific.
    
    var endAddress = @ + 16;

    if (isInt)
    {
        int iValue;
    }
    else
    {
        float floatValues[4];
    }

    @ = endAddress;
};
```

## `alignas`

All types in BEdit have a default alignment of 1, this mean no padding is inserted between members. This might cause issues when copy-pasting types from C.

Padding is compiler specific and cannot be assumed by BEdit, but you can add `alignas` specifier to align `@` before the member is evaluated.

```C++
typedef string(1) char;
typedef alignas(4) s(4) int;

struct Foo
{
    char c; // Start address is zero.
    int i;  // Start address is four.
    alignas(3) raw(1) funky; // Start address is 9.
};

layout Foo;
```

Note that alignment corrections take place just before the member, that is:

```C++
struct Foo
{
    u(1) uVal;
    print(@); // prints 1, not 4.
    alignas(4) s(4) iVal;
};
``` 

`alignas` can be used on any member declaration (except category member) and any type declaration. If both a type and member has declared an `alignas` the one declared for the member is used. If a type of a category has been declared with `alignas` all types of that category must have the same alignment.

## `import`

As files become bigger, separating them to several different ones is sometimes a good idea. You can use `import` for this.

```C++
// Defines u32 and MyString
import("MyTypes.bet");

struct Foo
{
    u32 uVal;
    MyString str;
};

layout Foo;
```

When a module is being imported, any `layout` command therein is ignored.

## External Members

In certain situations you have links to other parts of the file (e.g. `fileOffsetToPixelArray` in BMP example). Sometimes it's useful to "pretend" the data is in that type, for display reasons, but you don't want the struct size to be modified due to it.

Consider

```C++
struct Image
{
    u(4) width;
    u(4) height;
    u(8, hex) pixelsLocation;

    // Not possible as this modifies @
    // @(pixelsLocation) raw(4) rgba[width * height];
};

struct File
{
    u(4) imageCount;
    Image images[imageCount];

    // Collection of all pixels in the file.
    raw(size_of_file - @) allPixels;
};
```

To declare a member external, you can use:
```C++
    @(external pixelsLocation) raw(4) rgba[width * height];
```

An `external` member behaves exactly like a normal one except it resets `@` after it's declaration to what it was before. It is equivalent to 

```C++
    var oldAt = @;
    @(location) type member;
    @ = oldAt;
```

## `sizeof`

To get the size of a struct, `sizeof` can be used. The size of a type is defined as the difference between `@` before the type was evaluated and after.

As a type can vary in size depending on the current address (e.g. if it reads an array length from the binary file), you can also add an address specifier as `sizeof(@(address) MyStruct)`. If an address specifier is missing it uses the current address.

```C++
struct Bar
{
    u(1, hex) a;
    raw(a) b;
};

struct Foo
{
    print(sizeof(Bar));
    Bar bar;
    print(sizeof(Bar));
    print(sizeof(@(1) Bar));
};

layout Foo;

//
// NOTE: Expected output with binary file '01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F'
//
// "(sizeof(Bar)) 2"
// bar.a 1h
// bar.b 0x02
// "(sizeof(Bar)) 4"
// "(sizeof(@(1) Bar)) 3"
//
```

Currently only `struct` types are allowed in `sizeof` expressions.

## Future...

BEdit and the Layout Language is still in early development and several new features are planned. Some of the planned features are: functions, user defined encoded scalars, expressions using floating point arithmetics, non-scalar variables, binary format for "compiled" layout code... and much more.

For feature requests and bug reports, head over to [GitHub issue tracker](https://github.com/Kipt/BEdit/issues) or send a line directly to [bedit.jens@gmail.com](mailto://bedit.jens@gmail.com), all emails are treated as confidential. Forums are available at [Handmade Network](https://bedit.handmade.network/) along with the latest free version packaged for Windows and Linux.

To support development you can purchase the GUI version from [itch.io](https://kipt.itch.io/bedit) that not only allows you to view the binary file, but also comes with an editor.
