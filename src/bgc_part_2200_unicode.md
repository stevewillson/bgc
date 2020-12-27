<!-- Beej's guide to C

# vim: ts=4:sw=4:nosi:et:tw=72
-->

# Unicode, Wide Characters, and All That

Before we begin, note that this is an active area of language
development in C as it works to get past some, erm, _growing pains_.
When C2x comes out, updates here are probable.

Most people are basically interested in the deceptively simple question,
"How do I use such-and-such character set in C?" We'll get to that. But
as we'll see, it might already work on your system. Or you might have to
punt to a third-party library.

We're going to take about a lot of things this chapter---some are
platform agnostic, and some are C-specific.

Let's get an outline first of what we're going to look at:

* Unicode background
* Character encoding background
* Source and Execution character Sets
* Using Unicode and UTF-8
* Using other character types like `wchar_t`, `char16_t`, and `char32_t`

Let's dive in!

## What is Unicode?

Back in the day, it was popular in the US and much of the world to use a
7-bit or 8-bit encoding for characters in memory. This meant we could
have 128 or 256 characters (including non-printable characters) total.
That was fine for a US-centric world, but it turns out there are
actually other alphabets out there---who knew? Chinese has over 50,000
characters, and that's not fitting in a byte.

So people came up with all kinds of alternate ways to represent their
own custom character sets. And that was fine, but turned into a
compatibility nightmare.

To escape it, Unicode was invented. One character set to rule them all.
It extends off into infinity (effectively) so we'll never run out of
space for new characters. It has Chinese, Latin, Greek, cuniform, chess
symbols, emojis... just about everything, really! And more is being
added all the time!

## Code Points

I want to talk about two concepts here. It's confusing because they're
both numbers... different numbers for the same thing. But bear with me.

Let's loosely define _code point_ to mean a numeric value representing a
character. (Code points can also represent unprintable control
characters, but just assume I mean something like the letter "B" or the
character "π".)

Each code point represents a unique character. And each character has a
unique numeric code point associated with it.

For example, in Unicode, the numeric value 66 represents "B", and 960
represents "π". Other character mappings that aren't Unicode use
different values, potentially, but let's forget them and concentrate on
Unicode, the future!

So that's one thing: there's a number that represents each character. In
Unicode, these numbers run from 0 to over 1 million.

Got it?

Because we're about to flip the table a little.

## Encoding

If you recall, an 8-bit byte can hold values from 0-255, inclusive.
That's great for "B" which is 66---that fits in a byte. But "π" is 960,
and that doesn't fit in a byte! We need another byte. How do we
represent that in memory? Or what about bigger numbers, like 195,024?
That's going to need a number of bytes to hold.

How are these numbers represented in memory? This is what we call the
_encoding_ of the characters.

So we have two things: one is the code point which tells us effectively
the serial number of a particular character. And we have the encoding
which tells us how we're going to represent that number in memory.

There are plenty of encodings. You can make up your own right now, if
you want^[For example, we could store the code point in a big-endian
32-bit integer. Straightforward! We just invented an encoding! Actually
not; that's what UTF-32BE encoding is. Oh well---back to the grind!].
But we're going to look at some really common encodings that are in use
with Unicode.

|Encoding|Description|
|:-:|:-|
|UTF-8|A byte-oriented encoding that uses a variable number of bytes per character. This is the one to use.|
|UTF-16|A 16-bit per character^[Ish. Technically, it's variable width---there's a way to represent code points higher than $2^{16}$ by putting two UTF-16 characters together.] encoding.|
|UTF-32|A 32-bit per character encoding.|

With UTF-16 and UTF-32, the byte order matters, so you might see
UTF-16BE for big-endian and UTF-16LE for little-endian. Same for UTF-32.
Technically, if unspecified, you should assume big-endian. But since
Windows uses UTF-16 extensively and is little-endian, sometimes that is
assumed^[There's a special character called the _Byte Order Mark_ (BOM),
code point 0xFEFF, that can optionally precede the data stream and
indicate the endianess. It is not required, however.].

Let's look at some examples. I'm going to write the values in hex
because that's exactly two digits per 8-bit byte, and it makes it easier
to see how things are arranged in memory.

|Character|Code Point|UTF-16BE|UTF-32BE|UTF-16LE|UTF-32LE|UTF-8|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|`A`|41|0041|00000041|4100|41000000|41|
|`B`|42|0042|00000042|4200|42000000|42|
|`~`|7E|007E|0000007E|7E00|7E000000|7E|
|`π`|3C0|03C0|000003C0|C003|C0030000|CF80|
|`€`|20AC|20AC|000020AC|AC20|AC200000|E282AC|

Look in there for the patterns. Note that UTF-16BE and UTF-32BE are
simply the code point represented directly as 16- and 32-bit
values^[Again, this is only true in UTF-16 for characters that fit in
two bytes.].

Little-endian is the same, except the bytes are in little-endian order.

Then we have UTF-8 at the end. First you might notice that the
single-byte code points are represented as a single byte. That's nice.
You might also notice that different code points take different number
of bytes. This is a variable-width encoding.

So as soon as we get above a certain value, UTF-8 starts using
additional bytes to store the values. And they don't appear to correlate
with the code point value, either.

[flw[The details of UTF-8 encoding|UTF-8]] are beyond the scope of this
guide, but it's enough to know that it has a variable number of bytes
per code point, and those byte values don't match up with the code point
_except for the first 128 code points_.

That last bit is a neat thing about Unicode and UTF-8 from a North
American perspective: it's backward compatible with 7-bit ASCII
encoding! So if you're used to ASCII, UTF-8 is the same! Every
ASCII-encoded document is also UTF-8 encoded! (But not the other way
around, obviously.)

It's probably that last point more than any other that is driving UTF-8
to take over the world.

## Source and Execution Character Sets

When programming in C, there are (at least) three character sets that
are in play:

* The one that your code exists on disk as.
* The one the compiler translates that into just as compilation begins
  (the _source character set_). This might be the same as the one on
  disk, or it might not.
* The one the compiler translates the source character set into for
  execution (the _execution character set_). This might be the same as
  the source character set, or it might not.

Your compiler probably has options to select these character sets at
build-time.

The basic character set for both source and execution will contain
the following characters:

```
A B C D E F G H I J K L M
N O P Q R S T U V W X Y Z
a b c d e f g h i j k l m
n o p q r s t u v w x y z
0 1 2 3 4 5 6 7 8 9
! " # % & ' ( ) * + , - . / :
; < = > ? [ \ ] ^ _ { | } ~
space tab vertical-tab
form-feed end-of-line
```

Those are the characters you can use in your source and remain 100%
portable.

The execution character set will additionally have characters for alert
(bell/flash), backspace, carriage return, and newline.

But most people don't go to that extreme and freely use their extended
character sets in source and executable, especially now that Unicode and
UTF-8 are getting more common.

Notably, it's a pain (though possible with escape sequences) to enter
Unicode characters using only the basic character set.

## Unicode in C

Before I get into encoding in C, let's talk about Unicode from a code
point standpoint. There is a way in C to specify Unicode characters and
these will get translated by the compiler into the execution character
set^[Presumably the compiler makes the best effort to translate the code
point to whatever the output encoding is, but I can't find any
guarantees in the spec.].

So how do we do it?

How about the euro symbol, code point 0x20AC. (I've written it in hex
because both ways of representing it in C require hex.) How can we put
that in our C code?

Use the `\u` escape to put it in a string, e.g. `"\u20AC"` (case for the
hex doesn't matter). You must put **exactly four** hex digits after the
`\u`, padding with leading zeros if necessary.

Here's an example:

``` {.c}
char *s = "\u20AC1.23";

printf("%s\n", s);  // €1.23
```

So `\u` works for 16-bit Unicode code points, but what about ones bigger
than 16 bits? For that, we need capitals: `\U`.

For example:

``` {.c}
char *s = "\U0001D4D1";

printf("%s\n", s);  // Prints a mathematical letter "B"
```

It's the same as `\u`, just with 32 bits instead of 16. These are
equivalent:

``` {.c}
\u03C0
\U000003C0
```

Again, these are translated into the execution character set during
compilation. They represent Unicode code points, not any specific
encoding. Furthermore, if a Unicode code point is not representable in
the execution character set, the compiler can do whatever it wants with
it.

Now, you might wonder why you can't just do this:

``` {.c}
char *s = "€1.23";

printf("%s\n", s);  // €1.23
```

And you probably can, given a modern compiler. The source character set
will be translated for you into the execution character set by the
compiler. But compilers are free to puke out if they find any characters
that aren't included in their extended character set, and the € symbol
certainly isn't in the basic character set.

Caveat from the spec: you can't use `\u` or `\U` to encode any
code points below 0xA0 except for 0x24 (`$`), 0x40 (`@`), and 0x60
(`` ` ``).

Finally, you can also use these in identifiers in your code, with some
restrictions. But I don't want to get into that here. We're all about
string handling in this chapter.

And that's about it for Unicode in C (except encoding).

## A Quick Note on UTF-8 Before We Swerve into the Weeds

It could be that your source file on disk, the extended source
characters, and the extended execution characters are all in UTF-8
format. And the libraries you use expect UTF-8. This is the glorious
future of UTF-8 everywhere.

If that's the case, and you don't mind being non-portable to systems
that aren't like that, then just run with it. Stick Unicode characters
in your source and data at will. Use regular C strings and be happy.

A lot of things will just work, albeit non-portably. But maybe that's a
tradeoff that's worth it to you.

But there are some caveats.

* Things like `strlen()` report the number of bytes in a string, not the
  number of characters, necessarily.

* The following won't work properly with characters of more than one
  byte: `strtok()`, `strchr()` (use `strstr()` instead), `strspn()`-type
  functions, `toupper()`, `tolower()`, `isalpha()`-type functions, and
  probably more. Beware anything that operates on bytes.

* `printf()` variants allow for a way to only print so many bytes of a
  string^[With a format specifier like `"%s.12"`, for example.]. You
  want to make certain you print the correct number of bytes to end on a
  character boundary.

* If you want to `malloc()` space for a string, or declare an array of
  `char`s for one, be aware that the maximum size could be more than you
  were expecting. Each character could take up to `MB_LEN_MAX` bytes
  (from `<limits.h>`)---except characters in the basic character set
  which are guaranteed to be one byte.

And probably others I haven't discovered. Let me know what pitfalls
there are out there...

## Different Character Types

I want to introduce more character types. We're used to `char`, right?

But that's too easy. Let's make things a lot more difficult! Yay!

### Multibyte Characters

First of all, I want to potentially change your thinking about what a
string (array of `char`s) is. These are _multibyte strings_ made up of
_multibyte characters_.

That's right---your run-of-the-mill string of characters is multibyte.

Even if a particular character in the string is only a single byte, or
if a string is made up of only single characters, it's known as
multibyte.

For example:

``` {.c}
char c[128] = "Hello, world!";  // Multibyte string
```

What we're saying here is that a particular character that's not in the
basic character set could be composed of multiple bytes. Up to
`MB_LEN_MAX` of them (from `<limits.h>`). Sure, it only looks like one
character on the screen, but it could be multiple bytes.

You can throw Unicode values in there, as well, as we saw earlier:

``` {.c}
char *s = "\u20AC1.23";

printf("%s\n", s);  // €1.23
```

But here we're getting into some weirdness, because check this out:

``` {.c}
char *s = "\u20AC1.23";  // €1.23

printf("%zu\n", strlen(s));  // 7!
```

The string length of `"€1.23"` is `7`?! Yes! Well, on my system, yes!
Remember that `strlen()` returns the number of bytes in the string, not
the number of characters. (When we get to "wide characters", coming up,
we'll see a way to get the number of characters in the string.)`

Note that while C allows individual multibyte `char` constants, the
behavior of these varies by implementation and your compiler might warn
on it.

GCC, for example, warns of multi-character character constants for the
following two lines (and, on my system, prints out the UTF-8 encoding):

``` {.c}
printf("%x\n", '€');
printf("%x\n", '\u20ac');
```

### Wide Characters

If you're not a multibyte character, then you're a _wide character_.

A wide character is a single value that can uniquely represent any
character in the current locale. It's analogous to Unicode code points.
But it might not be. Or it might be.

Basically, where multibyte character strings are arrays of bytes, wide
character strings are arrays of _characters_. So you can start thinking
on a character-by-character basis rather than a byte-by-byte basis (the
latter of which gets all messy when characters start taking up variable
numbers of bytes).

Wide characters can be represented by a number of types, but the big
standout one is `wchar_t`. It's the main one.

You might be wondering if you can't tell if it's Unicode or not, how
does that allow you much flexibility in terms of writing code? `wchar_t`
opens some of those doors, as there are a rich set of function you can
use to deal with `wchar_t` strings (like getting the length, etc.)
without caring about the encoding.

## Using Wide Characters and `wchar_t`

Time for a new type: `wchar_t`. This is the main wide character typre.
Remember how a `char` is only one byte? And a byte's not enough to
represent all characters, potentially? Well, this one is enough.

To use `wchar_t`, `#include <wchar.h>`.

How many bytes big is it? Well, it's not totally clear. Could be 16
bits. Could be 32 bits.

But wait, you're saying---if it's only 16 bits, it's not big enough to
hold all the Unicode code points, is it? You're right---it's not. The
spec doesn't require it to be. It just has to be able to represent all
the characters in the current locale.

This can cause grief with Unicode on platforms with 16-bit `wchar_t`s
(ahem---Windows). But that's out of scope for this guide.

You can declare a string or character of this type with the `L` prefix,
and you can print them with the `%ls` ("ell ess") format specifier. Or
print an individual `wchar_t` with `%lc`.

``` {.c}
wchar_t *s = L"Hello, world!";
wchar_t c = L'B';

printf("%ls %lc\n", s, c);
```

Now---are those characters stored are Unicode code points, or not?
Depends on the implementation. But you can test if they are with the
macro `__STDC_ISO_10646__`. If this is defined, the answer is, "It's
Unicode!"

More detailedly, the value in that macro is an integer in the form
`yyyymm` that lets you know what Unicode standard you can rely
on---whatever was in effect on that date.

But how do you use them?

### Multibyte to `wchar_t` Conversions

So how do we get from the byte-oriented standard strings to the
character-oriented wide strings and back?

We can use a couple string conversion functions to make this happen.

First, some naming conventions you'll see in these functions:

* `mb`: multibyte
* `wc`: wide character
* `mbs`: multibyte string
* `wcs`: wide character string

So if we want to convert a multibyte string to a wide character string,
we can call the `mbstowcs()`. And the other way around: `wcstombs()`.

Let's do a quick demo where we convert a multibyte string to a wide
character string, and compare the string lengths of the two using their
respective functions.

``` {.c .numberLines}
#include <stdio.h>
#include <stdlib.h>
#include <wchar.h>
#include <string.h>
#include <locale.h>

int main(void)
{
    // Get out of the C locale to one that likely has the euro symbol
    setlocale(LC_ALL, "");

    // Original multibyte string with a euro symbol (Unicode point 20ac)
    char *mb_string = "The cost is \u20ac1.23";  // €1.23
    size_t mb_len = strlen(mb_string);

    // Wide character array that will hold the converted string
    wchar_t wc_string[128];  // Holds up to 128 wide characters

    // Convert the MB string to WC; this returns the number of wide chars
    size_t wc_len = mbstowcs(wc_string, mb_string, 128);

    // Print result--note the %ls for wide char strings
    printf("multibyte: \"%s\" (%zu bytes)\n", mb_string, mb_len);
    printf("wide char: \"%ls\" (%zu characters)\n", wc_string, wc_len);
}
```

On my system, this outputs:

```
multibyte: "The cost is €1.23" (19 bytes)
wide char: "The cost is €1.23" (17 characters)
```

(Your system might vary on the number of bytes depending on your
locale.)

One interesting thing to note is that `mbstowcs()`, in addition to
converting the multibyte string to wide, returns the length (in
characters) of the wide character string. And, in fact, it has a special
mode where it _only_ returns the length-in-characters of a given
multibyte string: you just pass `NULL` to the destination, and `0` to
the maximum number of characters to convert (this value is ignored).

(In the code below, I'm using my extended source character set---you
might have to replace those with `\u` escapes.)

``` {.c}
setlocale(LC_ALL, "");

// The following string has 7 characters
size_t len_in_chars = mbstowcs(NULL, "§¶°±π€•", 0));

printf("%zu", len_in_chars);  // 7
```

And, of course, if you want to convert the other way, it's `wcstombs()`.

## Wide Character Functionality

Once we're in wide character land, we have all kinds of functionality at
our disposal. I'm just going to summarize a bunch of the functions here,
but basically what we have here are the wide character versions of the
multibyte string functions that we're use to. (For example, we know
`strlen()` for multibyte strings; there's an `wcslen()` for wide
character strings.)

### I/O Functions

TODO output stream mode
TODO wint_t

Typically include `<stdio.h>` and `<wchar.h>` for these.

|I/O Function|Description|
|-|-|
|`wprintf()`|Formatted console output.|
|`wscanf()`|Formatted console input.|
|`getwchar()`|Character-based console input.|
|`putwchar()`|Character-based console output.|
|`fwprintf()`|Formatted file output.|
|`fwscanf()`|Formatted file input.|
|`fgetwc()`|Character-based file input.|
|`fputwc()`|Character-based file output.|
|`fgetws()`|String-based file input.|
|`fputws()`|String-based file output.|
|`swprintf()`|Formatted string output.|
|`swscanf()`|Formatted string input.|
|`vfwprintf()`|Variadic formatted file output.|
|`vfwscanf()`|Variadic formatted file input.|
|`vswprintf()`|Variadic formatted string output.|
|`vswscanf()`|Variadic formatted string input.|
|`vwprintf()`|Variadic formatted console output.|
|`vwscanf()`|Variadic formatted console input.|
|`ungetwc()`|Push a wide character back on an output stream.|
|`fwide()`|Get or set stream multibyte/wide orientation.|

### Type Conversion Functions

Typically include `<wchar.h>` for these.

|Conversion Function|Description|
|-|-|
|`wcstod()`|Convert string to `double`.|
|`wcstof()`|Convert string to `float`.|
|`wcstold()`|Convert string to `long double`.|
|`wcstol()`|Convert string to `long`.|
|`wcstoll()`|Convert string to `long long`.|
|`wcstoul()`|Convert string to `unsigned long`.|
|`wcstoull()`|Convert string to `unsigned long long`.|

### String and Memory Copying Functions

Typically include `<wchar.h>` for these.

|Copying Function|Description|
|----|----------------------------------------------|
|`wcscpy()`|Copy string.|
|`wcsncpy()`|Copy string, length-limited.|
|`wmemcpy()`|Copy memory.|
|`wmemmove()`|Copy potentially-overlapping memory.|
|`wcscat()`|Concatenate strings.|
|`wcsncat()`|Concatenate strings, length-limited.|

### String and Memory Comparing Functions

Typically include `<wchar.h>` for these.

|Comparing Function|Description|
|-------------------|---------------------------------------------------------------|
|`wcscmp()`|Compare strings lexicographically.|
|`wcsncmp()`|Compare strings lexicographically, length-limited.|
|`wcscoll()`|Compare strings in dictionary order by locale.|
|`wmemcmp()`|Compare memory lexicographically.|
|`wcsxfrm()`|Transform strings into versions such that `wcscmp()` behaves like `wcscoll()`[^97d0].|

[^97d0]: `wcscoll()` is the same as `wcsxfrm()` followed by `wcscmp()`.

### String Searching Functions

Typically include `<wchar.h>` for these.

|Searching Function|Description|
|-|-|
|`wcschr()`|Find a character in a string.|
|`wcsrchr()`|Find a character in a string from the back.|
|`wmemchr()`|Find a character in memory.|
|`wcsstr()`|Find a substring in a string.|
|`wcspbrk()`|Find any of a set of characters in a string.|
|`wcsspn()`|Find length of substring including any of a set of characters.|
|`wcscspn()`|Find length of substring before any of a set of characters.|
|`wcstok()`|Find tokens in a string.|

### Length/Miscellaneous Functions

Typically include `<wchar.h>` for these.

|Length/Misc Function|Description|
|-|-|
|`wcslen()`|Return the length of the string.|
|`wmemset()`|Set characters in memory.|
|`wcsftime()`|Formatted date and time output.|

### Character Classification Functions

Include `<wctype.h>` for these.

|Length/Misc Function|Description|
|-|-|
|`iswalnum()`|True if the character is alphanumeric.|
|`iswalpha()`|True if the character is alphabetic.|
|`iswblank()`|True if the character is blank (space-ish, but not a newline).|
|`iswcntrl()`|True if the character is a control character.|
|`iswdigit()`|True if the character is a digit.|
|`iswgraph()`|True if the character is printable (except space).|
|`iswlower()`|True if the character is lowercase.|
|`iswprint()`|True if the character is printable (including space).|
|`iswpunct()`|True if the character is punctuation.|
|`iswspace()`|True if the character is whitespace.|
|`iswupper()`|True if the character is uppercase.|
|`iswxdigit()`|True if the character is a hex digit.|
|`towlower()`|Convert character to lowercase.|
|`towupper()`|Convert character to uppercase.|

TODO: shift state, UTF

### `wint_t`

This is related to `wchar_t` in nature. It's an integer that can
represent all values in the extended character set, and also a special
end-of-file character, `WEOF`.

This is used by a number of single-character-oriented wide character
functions.

To use it, `#include <wchar.h>`.

## Character Type Conversions

* int mbtowc(wchar_t * restrict pwc, const char * restrict s, size_t n);
* int wctomb(char *s, wchar_t wchar);

* size_t mbstowcs(wchar_t * restrict pwcs, const char * restrict s, size_t n);
* size_t wcstombs(char * restrict s, const wchar_t * restrict pwcs, size_t n);

* size_t mbrtowc(wchar_t * restrict pwc, const char * restrict s, size_t n, mbstate_t * restrict ps);
* size_t wcrtomb(char * restrict s, wchar_t wc, mbstate_t * restrict ps);

* size_t mbrtowc(wchar_t * restrict pwc, const char * restrict s, size_t n, mbstate_t * restrict ps);
* size_t wcrtomb(char * restrict s, wchar_t wc, mbstate_t * restrict ps);

* size_t mbsrtowcs(wchar_t * restrict dst, const char ** restrict src, size_t len, mbstate_t * restrict ps);
* size_t wcsrtombs(char * restrict dst, const wchar_t ** restrict src, size_t len, mbstate_t * restrict ps);


* wint_t btowc(int c);
* int wctob(wint_t c);


* size_t mbrtoc16(char16_t * restrict pc16, const char * restrict s, size_t n, mbstate_t * restrict ps);
* size_t c16rtomb(char * restrict s, char16_t c16, mbstate_t * restrict ps);
* size_t mbrtoc32(char32_t * restrict pc32, const char * restrict s, size_t n, mbstate_t * restrict ps);
* size_t c32rtomb(char * restrict s, char32_t c32, mbstate_t * restrict ps);

## Unicode Encodings and C

### UTF-8

In general, C doesn't dictate what encoding your characters use. The
spec is _really_ flexible in this regard.

But if you are dealing with UTF-8 data in C, there's something about it
that makes it C-friendly. You know how C strings end with a NUL (`'\0'`)
character? Well, good news: that character won't show up in any UTF-8
strings! (It's a valid UTF-8 character, but it's the NUL character, just
like we're used to.)

Also, all the string functions in C just process bytes until they find a
NUL. So you can have UTF-8 encoded data in there and things like
`strcpy()` will still work. But you have to be careful---a lot of other
functions, like `strlen()`, might not behave as you expect.

You can tell C that you specifically want a string to be UTF-8 encoded,
and it'll do it for you. You can prefix a string with `u8`:

``` {.c}
char *s = u8"Hello, world!";

printf("%s\n", s);   // Hello, world!--if you can output UTF-8
```

Now, can you put Unicode characters in there?

``` {.c}
char *s = u8"€123";
```

Sure! If the extended source character set supports it. (gcc does.)

What if it doesn't? You can specify a Unicode character with `\u` for
16-bit hex code points and `\U` for 32-bit hex code points. (You can
always use `\U`, it's just a lot of zeros up front that you might not
need.)

The "€" symbol is code point `20AC` in hex, so we could write the above
with:

``` {.c}
char *s = u8"\u20AC";
```

Or this way with UTF-32:

``` {.c}
char *s = u8"\U000020AC";
```

You must use exactly 4 hex digits with `\u` and exactly 8 hex digits
with `\U`. The spec also has a restriction: you can't use them to encode
any code points below 0xA0 except for 0x24 (`$`), 0x40 (`@`), and 0x60
(`` ` ``).

### `char16_t` and `char32_t`

`char16_t` and `char32_t` are a couple other potentially wide character
types with sizes of 16 bits and 32 bits, respectively. Not necessarily,
because if they can't represent every character in the current locale,
they lose their wide character nature. But the spec refers them as "wide
character" types all over the place, so there we are.

These are here to make things a little more Unicode-friendly,
potentially.

To use, `#include <uchar.h>`.

You can declare a string or character of these types with the `u` and
`U` prefixes:

``` {.c}
char16_t *s = u"Hello, world!";
char16_t c = u'B';

char32_t *t = U"Hello, world!";
char32_t d = U'B';
```

Now---are values in these stored in UTF-16 or UTF-32? Depends on the
implementation.

But you can test to see if they are. If the macros `__STDC_UTF_16__` or
`__STDC_UTF_32__` are defined (to `1`) it means the types hold UTF-16 or
UTF-32, respectively.

If you're curious, and I know you are, the values, if UTF-16 or UTF-32,
are stored in the native endianess. That is, you should be to compare
them straight up to Unicode code point values:

``` {.c}
char16_t pi = u"\u03C0";  // pi symbol

#if __STDC_UTF_16__
pi == 0x3C0;  // Always true
#else
pi == 0x3C0;  // Probably not true
#endif
```