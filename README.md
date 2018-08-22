# SDN

**s**imple **d**ata **n**otation

** Note: This is not yet finalized, there may still be breaking changes **

The billionth human-readable data format. Derived from and very similiar to [edn](https://github.com/edn-format/edn), but more minimalistic. Also attempts to specify how to precisely deal with floating point numbers.

## General

SDN must be encoded as utf8.

The whitespace characters are `\n` (ASCII character 10) and ` ` (ASCII character 32).
Whitespace is ignored other than to separate elements.

An sdn file consists of exactly one element.

Any line starting with `;` outside of a string literal is considered whitespace.

## Elements

### nil

`nil` is an element, it is the single value of the unit type. It represents absence of information.

### Booleans

`true` and `false` are elements, they are the two values of the boolean type.

### Strings

```sdn
"A string, \\ \" \t \n \u11B3  "
```

(This description is mostly stolen from [TOML](https://github.com/toml-lang/toml#user-content-string))

Strings are surrounded by quotation marks. Any Unicode character may
be used except those that must be escaped: quotation mark, backslash, and the
control characters (U+0000 to U+001F, U+007F).

Escape sequences:

```
\t         - tab             (U+0009)
\n         - linefeed        (U+000A)
\"         - quote           (U+0022)
\\         - backslash       (U+005C)
\uXXXX     - unicode         (U+XXXX)
\UXXXXXXXX - unicode         (U+XXXXXXXX)
```

Any Unicode character may be escaped with the `\uXXXX` or `\UXXXXXXXX` forms.
The escape codes must be valid Unicode [scalar values](http://unicode.org/glossary/#unicode_scalar_value).

### Integers

An integer consists of one or more digits `0` - `9`, optionally prefixed by a `-`. No integer other than `0` itself may begin with a `0`.

`-0` is not a valid integer.

If an integer is suffixed by a `N`, it is an arbitrary precision integer. Else, integers outside the range of a 64 bit two's complement (smaller than -9223372036854775808 or larger than 9223372036854775807) are invalid.

### Floats
A float consists of an integer (without an `N` suffix), followed by a dot `.`,  followed by one or more digits `0` - `9`. It may be prefixed by a `-`. It may be suffixed by an exponent. An exponent is an `E`, optionally followed by a `-`, followed by one or more digits `0` - `9`.

`NaN`, `Infinity`, `-Infinity` are valid floats. `-0.0` and `0.0` designate two different floats. There is only a single `NaN` (no signaling, no sign bit, no payload, etc).

Floats are [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754) 64 bit floats. Any floats that can not be represented exactly must be [rounded nearest, tied to even](https://en.wikipedia.org/wiki/Rounding#Round_half_to_even).

### Rationals
A rational consists of an optional `-`, followed by one or more digits `0` - `9`, followed by a `/`, followed by one or more digits `0` - `9`. The denominator may not be `0`. No numerator other than `0` and no nominator may begin with a `0`.

A rational is an arbitrary precision rational number. The concrete representation is not important, i.e. `2/6` may be stored as `1/3` internally (and both are considered equal in the equality relation defined below).

### Symbols

Symbols represent identifiers, valid characters are alphanumeric characters and any of `# : / . * + ! - _ ? $ % & = < >`. A symbol may not start with a numeric character, and `nil`, `true`, `false`, `NaN`, `Infinity`, and `-Infinity` are not valid symbols.

When some characters can be parsed as either a number or a symbol, they must be parsed as a number.

### Lists

```sdn
(x 14)
```

An ordered sequence of elements, represented as `(`, optional whitespace, any number of elements, optional whitespace and `)`.

### Set

```sdn
#{
  a b c
}
```

An unordered collection of elements, represented as `#{`, optional whitespace, any number of pairs of elements, optional whitespace, and `}`. Each element may appear at most once.

### Map

```sdn
{
  a b
  c d
}
```

An unordered collection of key-value pairs, represented as `{`, optional whitespace, any number of pairs of elements, optional whitespace, and `}`. Each element may appear at most once.

## Equality

To enforce the uniqueness of set entries and map keys, there has to be an equality relation.

`nil` is equal to `nil`, `true` is equal to `true`, `false` is equal to `false`.

Two integers are equal if they consist of the same characters. 64 bit integers and arbitrary precision integers are never equal.

Two symbols are equal if they consists of the same characters.

Two strings are equal if they result in the same data after resolving escape sequences.

Two floats are equal if they result in the same IEEE 754 64 bit float after rounding. In particular, `NaN` is equal to `NaN`.

Two rationals are equal if they both describe the same rational number, e.g. `2/6` equals `1/3`.

Two lists are equal if they have the same length and all corresponding pairs of elements are equal.

Two sets are equal if they contain the same set of entries, independent of the order.

Two maps are equal if they contain the same set of pairs, independent of the order.

All other pairs of elements (in particular elements of different types) are unequal.

## Canonical encoding

There can be multiple encodings of some value that decode to that same value. In some settings (e.g. when working with content-addressable storage), this ambiguity is harmful. This section describes a canonical way of encoding data to prevent these problems.

The canonical encoding somewhat defeats the point of a human-readable format, since it normalizes away all whitespace. For this reason, it might be better to use a more efficient (and in the case of floats less painful) binary encoding. For this to work, there has to be a bijection between valid canonical sdn encodings and valid canonical binary encodings.

The sources of non-unique encodings are:

- leading and trailing whitespace of an sdn file (this includes comments)
- escape sequences in strings
- floats with exponents
- floats with rounding errors
- rationals with different denominators for the same number
- whitespace in lists, sets and maps (this includes comments)
- order of entries in sets and maps

The canonic sdn representation of some data is obtained as followed:

Omit all leading and trailing whitespace. Omit all whitespace that is not needed for separating elements in collections. Separate these elements by exactly one space character (ASCII 32).

In strings, do not use any escape sequences other than for quotation marks `"`, the backslash `\` and the control characters (U+0000 to U+001F, U+007F). Escape `"` as `\"`, `\` as `\\`, and control characters as `\uXXXX`.

Floats must be encoded such that the resulting string:

- rounds to the correct float
- has a `0` left of the decimal point
- has a nonzero digit after the decimal point
- includes the exponent
- is a shortest possible string satisfying these criteria
- if there are multiple shortest strings satisfying these criteria, chose the one with the smaller exponent
- if there are multiple shortest strings satisfying these criteria with the same exponent, chose the smallest number among them

For convenience, here are some links to float formatting algorithms: [Dragon4](https://lists.nongnu.org/archive/html/gcl-devel/2012-10/pdfkieTlklRzN.pdf), [grisu](https://www.cs.tufts.edu/~nr/cs257/archive/florian-loitsch/printf.pdf) and [errol](https://cseweb.ucsd.edu/~lerner/papers/fp-printing-popl16.pdf).

For rationals, use the denominator with the smallest possible absolute value.

For sets and maps, we define a total order on canonical elements. Sets must list their elements sorted from smallest to largest, and maps must sort from smallest to largest key. The order is defined as the reflexive (according to equality as defined above), transitive closure of the following relation `<`:

- `nil` < `false`
- `false` < `true`
- `true` < any 64 bit int
- any 64 bit int < any arbitrary precision int
- any arbitrary precision int < any float
- any float < any rational
- any rational < any string
- any string < any symbol
- any symbol < any list
- any list < any set
- any set < any map
- for 64 bit ints `a` and `b`: `a` < `b` if a is smaller than b
- for arbitrary precision ints `a` and `b`: `a` < `b` if a is smaller than b
- for floats `a` and `b`: `a` < `b` if `totalOrder(a, b)` is true, as defined in the IEEE 754 standard. Which, by the way, sits behind a paywall, but searching the web for `IEEE 754 pdf` or something similiar helps.
- for rationals `a` and `b`: `a` < `b` if a is smaller than b
- for strings `a` and `b`: `a` < `b` if the utf8 encoding of a is lexicographically smaller than that of b
- for symbols `a` and `b`: `a` < `b` if the utf8 encoding of a is lexicographically smaller than that of b
- for lists `a` and `b` whose entries are already canonical:
  - if `a` is empty and `b` is not: `a` < `b`
  - if the first entry in `a` is smaller than the first entry in `b`: `a` < `b`
  - else, `a` < `b` if (`a` without its first entry) < (`b` without its first entry)
- for sets `a` and `b` whose entries are already canonical:
  - if `a` is empty and `b` is not: `a` < `b`
  - if the smallest entry in `a` is smaller than the smallest entry in `b`: `a` < `b`
  - else, `a` < `b` if (`a` without its smallest entry) < (`b` without its smallest entry)
- for maps `a` and `b` whose entries are already canonical:
  - if `a` is empty and `b` is not: `a` < `b`
  - if the smallest key in `a` is smaller than the smallest key in `b`: `a` < `b`
  - if the smallest key in `a` has a value smaller than the value associated with the smallest key in `b`: `a` < `b`
  - else, `a` < `b` if (`a` without its smallest key) < (`b` without its smallest key)
