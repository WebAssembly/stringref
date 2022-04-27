# Reference-Typed Strings

A minimum viable proposal to add a reference-typed strings to
WebAssembly (the `stringref` proposal).

## Champions

Andy Wingo `<wingo@igalia.com>`

## Goals

 1. Enable programs compiled to WebAssembly to efficiently create and
    consume JavaScript strings
 2. Provide a good string implementation that many languages implemented
    on top of the GC proposal would find useful

These goals are sometimes in tension!  The operative words to help us
find good compromises are "minimal" and "viable".

## Requirements

 1. Zero-copy passing of strings from JavaScript to WebAssembly & back
 2. No new string implementations on the web: allow re-use of JS
    engine's strings
 3. Allow WebAssembly implementations to efficiently represent strings
    internally in either WTF-8 or WTF-16 encodings
 4. Allow access to WTF-16 code units for Java, Dart, Kotlin and similar languages
 5. Allow string literals in element sections

## Definitions
 - *codepoint*: An integer in the range [0,0x10FFFF].
 - *surrogate*: A codepoint in the range [0xD800,0xDFFF].
 - *unicode scalar value*: A codepoint that is not a surrogate.
 - *character*: An imprecise concept that we try to avoid in this
    document.
 - *code unit*: An indivisible unit of an encoded unicode scalar value.
    For UTF-8 encodings, an integer in the range [0,0xFF] (a byte); for
    UTF-16 encodings, an integer in the range [0,0xFFFF]; for UTF-32,
    the unicode scalar value itself.
 - *high surrogate*: A surrogate in the range [0xD800,0xDBFF].
 - *low surrogate*: A surrogate which is not a high surrogate.
 - *surrogate pair*: A sequence of a *high surrogate* followed by a *low
    surrogate*, used by UTF-16 to encode a codepoint in the range
    [0x10000,0x10FFFF].
 - *isolated surrogate*: Any surrogate which is not part of a surrogate
    pair.

## Design

### What's a string?

Good question!  It sure would be nice to say that a string is a sequence
of unicode scalar values.  However, to satisfy the goal of being a good
compilation target for a wide range of programming languages, as well as
the goal of good JavaScript interoperability, things are a little more
complicated.

Some languages present no problem to this idea that strings are composed
of unicode scalar values.  Python and Rust are in this category.

Other languages, notably JavaScript and Java, define their strings to be
sequences of 16-bit code units, for historical reasons.  These code unit
sequences aren't quite UTF-16, because they can contain isolated
surrogates, which are prohibited by standard Unicode encoding forms.
The facility we end up building should be able to represent all Java
strings, as a compilation target, and also represent all JavaScript
strings, for good web iteroperability.

Therefore **we define a string to be a sequence of unicode scalar values
and isolated surrogates**.  The code units of a Java or JavaScript
string can be interpreted to encode such a sequence, in the
[WTF-16](https://simonsapin.github.io/wtf-8/) encoding form.

This proposal does not require a WebAssembly implementation to use
WTF-16 to represent its strings internally.  It only requires that the
implementation be able to represent all codepoint sequences that can be
encoded with WTF-16, notably codepoint sequences containing isolated
surrogates.

### Encodings and views

There is an impedance-matching problem between the way that WebAssembly
implementations want to represent strings, and the way that source
languages want to access strings.  The `stringref` API has to do the
best it can to smooth over these differences.

On the implementation side, some WebAssembly implementations will want
to represent string contents in the WTF-16 encoding, notably
implementations embedded on the web because of JavaScript's usage of
WTF-16.  Other implementations without legacy requirements will want to
use WTF-8, as it is generally more space-efficient and closer to the
common UTF-8 interchange format.

From the source language side, some source languages such as Java will
also want to consider strings as WTF-16 code unit sequences.  Other
source languages will want to think of strings as WTF-8 byte sequences,
and others will want to think of strings as codepoint sequences.

On the most basic level, when a source language wants to access string
contents, it can encode the whole string to memory (or eventually to a
GC-managed array) as UTF-8, WTF-8, or WTF-16, depending on the source
language's needs.  For example Java usually wants to treat strings as
WTF-16 code unit sequences, so Java would encode to WTF-16.  This
proposal provides facilities for measuring how many bytes an encoding
would take, and for actually doing the encoding.

This proposal also includes the ability to get a WTF-8 or WTF-16 "view"
on a string, which should provide near-constant-time random access to
the bytes of a WTF-8 encoding of the string, or to the 16-bit code units
of a WTF-16 encoding of a string.  We also provide a view that allows an
iterator interface over the codepoints in a string.

An implementation using WTF-8 will have some costs for source languages
that want WTF-16, and vice versa.

Getting a view for a WebAssembly implementation's "native" string
encoding is likely a constant-time operation, and possibly even free.
For example, getting a WTF-8 view for an implementation that uses WTF-8
to represent strings will be free.  Getting a WTF-16 view on a WTF-8
implementation could imply a copy, or possibly the computation of a
[breadcrumbs](https://www.swift.org/blog/utf8-string/#breadcrumbs)
table.

This proposal defines positions in strings only with respect to a
specific string view type.  Any interface that needs to refer to a
position in a string should take a string view and an offset whose
meaning depends on the string view type: a byte offset for a WTF-8
string view, a code unit offset for a WTF-16 view, or a codepoint offset
for a codepoint view.

### Handling invalid UTF-8

WTF-8 and WTF-16 are not interchange formats: they should not be used
when communicating data over a network, for example.  If a program goes
to encode a string to UTF-8, and the string contains an isolated
surrogate, the program can trap, or it can replace the codepoint with
`U+FFFD` (the replacement character).  The stringref facility provides
an efficient mechanism for detecting strings which are not valid USV
sequences.  When encoding data as UTF-8, it allows the programmer to
specify whether to replace any isolated surrogate or whether to trap.

For some source languages, defining strings to be sequences of unicode
scalar values and isolated surrogates is an antifeature: those source
languages do not require the ability to represent isolated surrogates
and would prefer to not be given strings with isolated surrogates.  We
sympathize.  On a boundary where such a program might receive a string
containing isolated surrogates, the program can check for such strings
and trap using facilities defined in this proposal.

### Prior discussions

 * The component model subgroup chose to agree that on component
   boundaries, strings consist of sequences of unicode scalar values:
   https://docs.google.com/presentation/d/1qVbBsDFmremBGVKiOAzRk7svjinNq6LXfJ1DzeFwKtc
   *The CG discussion and decision inform but don't constrain this proposal.*

 * The AssemblyScript developers floated a "universal strings" proposal
   which explicitly provided for the WTF-8 and WTF-16 encodings that can
   support unpaired surrogates:
   https://github.com/AssemblyScript/universal-strings
   *An excellent early draft which seriously tackles the WTF-16 problem.*

## Proposal

This proposal consists of a basic `stringref` facility and a (possibly
post-MVP) `stringview` facility.

The `stringref` facility defines a new reference type, `stringref`.
Literal `stringref` values can be embedded in a WebAssembly module, with
their contents taken from a new section.  WebAssembly programs can also
create `stringref` values from data encoded in memory or GC arrays in
the WTF-8 or WTF-16 encodings, and can likewise write `stringref`
contents to memory in these encodings.  There is an instruction to
concatenate `stringref` values.  Finally, `stringref` values can be
compared for equality.

The `stringview` facility allows WebAssembly to obtain a "view" on the
contents of a `stringref`, treating it as a sequence of values in the
WTF-8 and WTF-16 encodings, as well as treating a string as a sequence
of codepoints.  WebAssembly programs can use stringviews to encode parts
of strings to memory, access string contents by index, and to take
substrings.

## The `stringref` facility

One new reference type: `stringref`.  Opaque, like `externref` and
`funcref`.

When reading or writing encoded bytes, the address in memory at which to
read or write the bytes depends on the memory model of the WebAssembly
module.
```
address ::= i32 | i64
```
Such instructions also take the memory to which to read or write as an
immediate.

### Creating strings

```
(string.new_wtf8 $memory ptr:address bytes:i32)
  -> str:stringref
```
Create a new string from the *`bytes`* WTF-8 bytes in memory at *`ptr`*.
Out-of-bounds access will trap.  Attempting to create a string with
invalid WTF-8 will trap.  The maximum value for *`bytes`* is
2<sup>31</sup>–1; passing a higher value traps.

```
(string.new_wtf16 $memory ptr:address codeunits:i32)
  -> str:stringref
```
Create a new string from the *`codeunits`* code units encoded in memory at
*`ptr`*.  Out-of-bounds access will trap.  *`ptr`* must be two-byte
aligned, and will trap otherwise.  The maximum value for *`codeunits`*
is 2<sup>30</sup>–1; passing a higher value traps.

#### `string.new` size limits

Creating a string is a form of dynamic allocation and can fail.  The
same implementation running on different machines can have different
behaviors.  The specification can only say that byte/code-unit sizes
above a certain limit *must* fail; but for sizes within the limits, the
allocations *may* fail.  If an allocation fails, the implementation must
trap.  Fallible `string.new` is a possible future extension.

### String literals

```
(string.const contents:i32)
  -> str:stringref
```
Create a new string from the literal string *`contents`*, as in
`(string.const "Hello, World!")`.  This instruction is constant and can
be used in global variable initializers.

#### String literal section

The `string.const` section indicates the literal as an `i32` index into
a new custom section: a string table, encoded as a `vec(vec(u8))` of
valid WTF-8 strings.  Because literal strings can contain codepoint 0,
strings in the string table do not use NUL as a terminator. The string
table section must immediately precede the global section, or where the
global section would be, in the binary.

#### `string.const` size limits

The maximum size for the WTF-8 encoding of an individual string literal
is 2<sup>31</sup>–1 bytes.  Embeddings may impose their own limits which
are more restricted.  But similarly to `string.new`, instantiating a
module with string literals may fail due to lack of memory resources,
even if the string size is formally within the limits.  However
`string.const` itself never traps when passed a valid literal offset.

### Accessing string contents

All parameters and return values measuring a number of codepoints or a
number of code units represent these sizes as unsigned values.

```
(string.measure_utf8 str:stringref)
  -> bytes:i32
(string.measure_wtf8 str:stringref)
  -> bytes:i32
(string.measure_wtf16 str:stringref)
  -> bytes:i32
```
Measure how many bytes would be necessary to encode the contents of the
string *`str`* to UTF-8, WTF-8, or WTF-16 respectively.  For
`string.measure_utf8`, if the string contains an isolated surrogate,
return -1.

The maximum number of bytes returned by these instructions is
2<sup>31</sup>-1.  If an encoding would require more bytes, it is as if
the contents can't be encoded at all (the return value is -1).

```
wtf8_policy ::= 'utf8' | 'wtf8' | 'replace'
(string.encode_wtf8 $memory $wtf8_policy str:stringref ptr:address)
(string.encode_wtf16 $memory str:stringref ptr:address)
```
Encode the contents of the string *`str`* as UTF-8, WTF-8, or WTF-16,
respectively, to memory at *`ptr`*.  The number of bytes written will be
the same as returned by the corresponding `string.measure_*encoding*`.
Note that no `NUL` terminator is ever written.  For
`string.encode_utf8`, trap if the string contains an isolated surrogate.

The maximum number of bytes that can be encoded at once by
`string.encode` is 2<sup>31</sup>-1.  If an encoding would require more
bytes, it is as if the codepoints can't be encoded (a trap).

For `string.encode_wtf8`, if an isolated surrogate is seen, the behavior
determines on the *`$wtf8_policy`* immediate.  For `utf8`, trap.  For
`wtf8`, the surrogate is encoded as per WTF-8.  For `replace`, `U+FFFD`
(the replacement character) is encoded instead.  Note that the UTF-8
encoding of `U+FFFD` is the same length as the encoding of an isolated
surrogate.

### Concatenation

```
(string.concat a:stringref b:stringref) -> stringref
```
Return a new stringref containing the codepoints from *`a`* followed by
the codepoints from *`b`*.

Note that the implementation should take care when, at any future time,
treating the resulting string as a sequence of codepoints.  If *`a`*'s
last codepoint is a high surrogate and *`b`*'s first codepoint is a low
surrogate, these two codepoints combine into one, as if they were the
two code units of a UTF-16-encoded unicode scalar value.

Concatenating two strings is a form of dynamic allocation and can fail.
If an allocation fails, the implementation must trap.  Fallible
`string.concat` is a possible future extension.

### Predicates

```
(string.eq a:stringref b:stringref) -> i32
```
Return 1 if the strings *`a`* and *`b`* contain the same codepoint
sequence.  Return 0 otherwise.

```
(string.is_usv_sequence str:stringref)
  -> bool:i32
```
Return 1 if the string *`str`* is a sequence of unicode scalar values,
and 0 otherwise.  A 0 result indicates that *`str`* contains isolated
surrogates.

## The `stringview` facility

Three new reference types: `stringview_wtf8`, `stringview_wtf16`, and
`stringview_iter`.  Opaque, like `externref` and `funcref`.

### `stringview_wtf8`

```
(string.as_wtf8 str:stringref)
  -> view:stringview_wtf8
```
Obtain a view on a string's contents as WTF-8.  The stringview can then
be used to interpret the string as a byte sequence in the WTF-8
encoding.

```
(stringview_wtf8.advance view:stringview_wtf8 pos:i32 bytes:i32)
  -> next_pos:i32
```
Starting at offset *`pos`* into the WTF-8 encoding of *`view`*, return
the highest offset that is not greater than *`pos` + `bytes`*.

If *`pos`* is greater than the WTF-8 byte length of *`view`*, it is
as if it were instead given as the byte length.  If *`pos`* is less than
the byte length but does not indicate the byte offset of the start of a
codepoint, it is advanced to the next codepoint (or the end of the
string, for the last codepoint).  Collectively these transformations are
the "WTF-8 position treatment".

If the mathematical value of *`next_pos`* would be greater than
2<sup>31</sup>, trap.  (Future extensions of the `stringref` proposal
along the lines of the [memory64
proposal](https://github.com/WebAssembly/memory64/blob/main/proposals/memory64/Overview.md)
may allow for 64-bit variants of the position-using instructions, which
could relax this restriction.)

```
(stringview_wtf8.encode $memory $wtf8_policy view:stringview_wtf8 ptr:address pos:i32 bytes:i32)
  -> next_pos:i32, bytes:i32
```
Write a subsequence of the WTF-8 encoding of *`view`* to memory at
*`ptr`*, starting at the WTF-8 offset *`pos`*, writing no more than
*`bytes`* bytes.  No NUL byte is written.  Return the WTF-8 offset of
the next characters to encode, along with the number of bytes written.

*`pos`* receives the "WTF-8 position treatment", as for
`stringview_wtf8.advance`.

If the mathematical value of *`next_pos`* would be greater than
2<sup>31</sup>, trap.  (Future extensions of the `stringref` proposal
along the lines of the [memory64
proposal](https://github.com/WebAssembly/memory64/blob/main/proposals/memory64/Overview.md)
may allow for 64-bit variants of the position-using instructions, which
could relax this restriction.)

If an isolated surrogate is seen, the behavior determines on the
*`$wtf8_policy`* immediate, as in `string.encode_wtf8`.

```
(stringview_wtf8.slice view:stringview_wtf8 start:i32 end:i32)
  -> str:stringref
```
Return a substring of *`view`*, for the WTF-8 bytes starting at offset
*`start`* and not greater than *`end`*.  *`start`* and *`end`* receive
the "WTF-8 position treatment", as for `stringview_wtf8.advance`.

### `stringview_wtf16`

```
(string.as_wtf16 str:stringref)
  -> view:stringview_wtf16
```
Obtain a view on a string's contents as WTF-16.  The stringview can then
be used to interpret the string as a sequence of 16-bit code units in
the WTF-16 encoding.

```
(stringview_wtf16.length view:stringview_wtf16)
  -> length:i32
```
Return the total number of 16-bit code units necessary to represent
*`view`* in the WTF-16 encoding.

```
(stringview_wtf16.get_codeunit view:stringview_wtf16 pos:i32)
  -> codeunit:i32
```
Return the 16-bit code unit at offset *`pos`* in the WTF-16 encoding of
*`view`*.  If *`pos`* is greater than or equal to the WTF-16 length of
*`view`*, trap.

```
(stringview_wtf16.encode $memory view:stringview_wtf16 ptr:address pos:i32 len:i32)
```
Write a subsequence of the WTF-16 encoding of *`view`* to memory at
*`ptr`*, starting at the WTF-16 offset *`pos`*, writing no more than
*`len`* 16-bit code units.  If *`ptr`* is not two-byte aligned, trap.

If *`pos`* is greater than the number of WTF-16 code units in *`view`*,
it is as if it were instead given as the code unit length.  This
transformation is the "WTF-16 position treatment".

```
(stringview_wtf16.slice view:stringview_wtf16 start:i32 end:i32)
  -> str:stringref
```
Return a substring of *`view`*, for the WTF-16 code units starting at offset
*`start`* and not greater than *`end`*.  *`start`* and *`end`* receive
the "WTF-16 position treatment", as for `stringview_wtf16.encode`.

### `stringview_iter`

```
(string.as_iter str:stringref)
  -> view:stringview_iter
```
Obtain a view on a string's contents as an iterator over the codepoints
in *`str`*, initially positioned at the beginning of the string.  The
stringview can then be used to iterate over the codepoints of the
string.

```
(stringview_iter.cur view:stringview_iter)
  -> codepoint:i32
```
Return the codepoint currently pointed to by *`view`*, or -1 if the
iterator is at the end of the string.

```
(stringview_iter.advance view:stringview_iter codepoints:i32)
  -> codepoints:i32
```
Advance the iterator *`view`* by up to *`codepoints`* codepoints.
Return the number of codepoints that were actually consumed.

```
(stringview_iter.rewind view:stringview_iter codepoints:i32)
  -> codepoints:i32
```
Rewind the iterator *`view`* by up to *`codepoints`* codepoints.
Return the number of codepoints that were actually consumed.

```
(stringview_iter.slice view:stringview_iter codepoints:i32)
  -> str:stringref
```
Return a substring of *`view`*, starting at the current position of
*`view`* and continuing for at most *`codepoints`* codepoints.

## Examples

We assume that the textual syntax for `string.encode` and `string.new`
allows you to elide the memory, in which case it defaults to 0.

### Make string from NUL-terminated UTF-8 in memory

```wasm
(func $string-from-utf8 (param $ptr i32) (result stringref)
  local.get $ptr
  local.get $ptr
  call $strlen
  string.new_wtf8)
```

Generally speaking, this proposal only distinguishes between UTF-8 and
WTF-8 when encoding string contents to memory.  As this is a a decode
operation, the proposal just has a WTF-8 interface, as WTF-8 is a
superset of UTF-8.

### Make string from an array of WTF-8 code units in memory

```wasm
(func $string-from-wtf8n (param $ptr i32) (param $len i32) (result stringref)
  local.get $ptr
  local.get $len
  string.new_wtf8)
```

### Make string from UTF-16 in memory

```wasm
(func $string-from-utf16 (param $ptr i32) (param $units i32) (result stringref)
  local.get $ptr
  local.get $units
  string.new_wtf16)
```

This proposal doesn't distinguish between UTF-16 and WTF-16 at all;
rather it just deals in WTF-16, as most source languages that expose
16-bit code units to users actually expose WTF-16 strings.

### Number of codepoints in string

```wasm
(func $codepoint-length (param $str stringref) (result i32)
  local.get $str
  string.as_iter      ;; Get iterator view
  i32.const -1        ;; advance by all codepoints
  stringview_iter.advance) ;; return number of codepoints advanced
```

### String literals

```wasm
(global $hey stringref (string.const "Hey"))

(func $howdy (result stringref)
  (string.const "Howdy"))

(func $is-cowboy (param $str stringref) (result i32)
  local.get $str
  call $howdy
  string.eq)
```

### Return the first codepoints of a string, as a stringref

```wasm
(func $prefix (param $str stringref) (param $codepoints i32)
              (result stringref)
  local.get $str
  string.as_iter
  local.get $codepoints
  stringview_iter.slice)
```

### Return a slice of WTF-16 code units of a string, as a stringref

```wasm
(func $slice (param $str stringref)
             (param $offset i32) (param $codeunits i32)
             (result stringref)
  local.get $str
  string.as_wtf16
  local.get $offset
  local.get $codeunits
  stringview_wtf16.slice)
```

### Suffix, prefix comparisons

There are a few ways to compare against a substring, but the easiest is
probably to slice the string, which is something you can do only with
respect to a particular encoding and view.  Given that we're comparing
against known strings, we know how long of a slice to take.

```wasm
(func starts-with-hey? (param $str stringref) (result i32)
  local.get $str
  string.as_wtf8
  i32.const 0
  i32.const 3
  stringview_wtf8.slice
  global.get $hey
  string.eq)

(func ends-with-howdy?/wtf8 (param $str stringref) (result i32)
  (local $wtf8 stringview_wtf8)

  local.get $str
  string.as_wtf8
  local.set $wtf8

  local.get $wtf8
  local.get $wtf8
  ;; Get wtf-8 offset of end
  i32.const 0
  i32.const -1
  stringview_wtf8.advance
  ;; Subtract 5.  Given WTF-8 position treatment, OK to wrap or 
  ;; not be on codepoint boundary.
  i32.const 5
  i32.sub
  ;; Slice until end.  If string ends with "howdy", these will be
  ;; 5 1-byte codepoints.
  i32.const -1
  stringview_wtf8.slice

  string.const "Howdy"
  string.eq)

;; WTF-16 flavor is similar.
(func ends-with-howdy?/wtf16 (param $str stringref) (result i32)
  (local $wtf16 stringview_wtf16)

  local.get $str
  string.as_wtf16
  local.set $wtf16

  ;; Slice last 5 code units.
  local.get $wtf16
  local.get $wtf16
  stringview_wtf16.length
  i32.const 5
  i32.sub
  i32.const -1
  stringview_wtf16.slice

  string.const "Howdy"
  string.eq)

;; Finally, a version with the iterator API.
(func ends-with-howdy?/iter (param $str stringref) (result i32)
  (local $iter stringview_iter)

  local.get $str
  string.as_iter
  local.set $iter

  ;; Advance to end.
  local.get $iter
  i32.const -1
  stringview_iter.advance

  ;; Rewind by 5.
  local.get $iter
  i32.const 5
  stringview_iter.rewind

  ;; Slice.
  local.get $iter
  i32.const 5
  stringview_iter.slice

  ;; Compare.
  string.const "Howdy"
  string.eq)
```

Which version of `ends-with-howdy?` will a source language produce?
They are essentially equivalent in this use case of comparing against a
static string, but in the general case, a source language that processes
strings in terms of codepoints would probably use the iterator,
languages that treat strings as UTF-8 sequences would produce the WTF-8
version whereas those that process strings in terms of 16-bit code units
will compile to the WTF-16 version.

One could instead do a character-by-character comparison, to avoid
creating the slice.

Stepping back a bit, prefix and suffix checks are examples of operations
for which the stringref proposal should facilitate high-performance
implementations.  The primary strategy of the stringref proposal is to
allow any such operation to be build in terms of its primitives.
However if there are important compound operations (e.g. prefix/suffix
checks) that can be sped up with a dedicated instruction, we should be
open to considering adding more instructions.

### Store a `stringref` without copying

```wasm
(table $strings 100 stringref)
(global $next-handle i32 (i32.const 0))

(func $intern-string (param $str stringref) (result i32)
  (local $handle i32)
  global.get $next-handle
  local.tee $handle
  local.get $str
  table.set $strings
  i32.const 1
  i32.add
  global.set $next-handle
  local.get $handle)
```

### Copy string contents to application-managed memory

```wasm
(func $malloc (param i32) (result i32))
(func $utf8-contents (param $str stringref) (result i32)
  (local $cur i32)
  (local $len i32)
  (local $ptr i32)
  local.get $str
  string.measure_utf8
  local.set $len

  block $valid
    local.get $len
    i32.const -1
    i32.ne
    br_if $valid
    unreachable                    ;; trap on error
  end

  local.get $len
  i32.const 1
  i32.add
  call $malloc                     ;; reserve space for bytes and NUL
  local.set $ptr

  local.get $str
  local.get $ptr
  string.encode_wtf8

  local.get $ptr
  local.get $len
  i32.add
  i32.const 0
  i32.store8                       ;; write NUL

  local.get $ptr
  return)
```

Using `string.measure_utf8` ensures that the encoded string is a valid
unicode scalar value sequence.  How to handle invalid UTF-8 is up to the
user; instead of `unreachable` we could throw an exception.

If we meant to handle isolated surrogates, we could use
`string.measure_wtf8` instead.

### Stream over contents of string

Assume you have a 1024-byte array of memory at `$buf`.  This function
will encode isolated surrogates as WTF-8.

```wasm
(global $buf i32)
(func $process-wtf8 (param $ptr i32) (param $len i32))

(func $process-string (param $str stringref)
  (local $cursor i32)                ;; initial value of 0 is start
  (local $bytes i32)

  loop
    local.get $str
    local.get $cursor
    global.get $buf
    i32.const 1024
    string.encode_wtf8               ;; push bytes written
    local.tee $bytes
    (if i32.eqz (then return))       ;; if no bytes encoded, done
    local.get $bytes
    local.get $cursor
    i32.add
    local.set $cursor

    global.get $buf
    local.get $bytes
    call $process-utf8
  end)
```

### Stream over UTF-16 code units of string, handling isolated surrogates

This function is probably slower than encoding chunks of the string to
WTF-16 in linear memory, for longer strings.

```wasm
(func $have-code-unit (param $codeunit i32))

(func $process-string (param $str stringref)
  (local $wtf16 stringview_wtf16)
  (local $cur i32)
  (local $len i32)

  local.get $str
  string.as_wtf16
  local.set $wtf16

  local.get $wtf16
  stringview_wtf16.length
  local.set $len

  block $done
    loop $loop
      local.get $cur
      local.get $len
      i32.ge
      br_if $done

      local.get $wtf16
      local.get $cur
      stringview_wtf16.get_codeunit
      call $have-code-unit

      i32.const 1
      local.get $cur
      i32.add
      local.set $cur
    end
  end)
```

### Stream over codepoints of string, handling isolated surrogates

This function is probably slower than encoding chunks of the string to
WTF-8 in memory, for longer strings.

```wasm
(func $have-codepoint (param $codepoint i32))

(func $process-string (param $str stringref)
  (local $iter stringview_iter)
  (local $ch i32)

  local.get $str
  string.as_iter
  local.set $iter

  block $done
    loop $loop
      local.get $iter
      stringview_iter.cur
      local.tee $ch

      i32.const -1
      i32.eq
      br_if $done

      local.get $ch
      call $have-codepoint

      local.get $iter
      i32.const 1
      string.advance_wtf8
      drop
    end
  end)
```

### Concatenate two strings

```wasm
(func $append (param $a stringref) (param $b stringref)
              (result stringref)
  local.get $a
  local.get $b
  string.concat)
```

## FAQ

### What does Emscripten currently do for strings and how could WebAssembly strings help?

Generally speaking, Emscripten [eagerly converts JavaScript strings to
NUL-terminated
UTF-8](https://github.com/emscripten-core/emscripten/blob/main/src/preamble.js#L91-L100),
allocating space for the UTF-8 encoding in linear memory using stack
allocation.  The [stringToUTF8
function](https://github.com/emscripten-core/emscripten/blob/main/src/runtime_strings.js#L158)
is written in JavaScript and handles surrogate pairs.  However for
isolated surrogates, [emscripten's decoder appears to produce
garbage](https://github.com/emscripten-core/emscripten/issues/15324).

For C functions that return strings, emscripten [parses NUL-terminated
UTF-8 from
memory](https://github.com/emscripten-core/emscripten/blob/main/src/preamble.js#L109),
[either using TextDecoder or via hand-rolled
JavaScript](https://github.com/emscripten-core/emscripten/blob/main/src/runtime_strings.js#L35).
Presumably TextDecoder is significantly faster as it doesn't have to
build rope strings.

Memory management is an issue, of course; the memory for a returned
string value may or may not be owned by the caller.

This proposal avoids memory ownership issues entirely, via automatic
memory management (implemented either via GC or reference counting).  It
also avoids eager string encoding onto the stack and the need for NUL
termination, allowing string contents to be written to memory exactly
where they are needed.

### Why should WebAssembly strings be able to represent isolated surrogates?

The main motivation is to support source languages with WTF-16 strings
(e.g. Java, Kotlin, C#).  JVM-based and CLR-based languages treat
strings as sequences of 16-bit code units.  Sometimes programs written
in e.g. Java will decode these sequences into codepoints encoded as
WTF-16, but not always.  Many common algorithms can be performed
directly on the code units, for example prefix matching.  Therefore to
efficiently support Java and friends when compiled to WebAssembly, we
need to support this view of strings as sequences of any 16-bit code
units, without any validity constraints that enforce that surrogates
always be properly paired.  This is the main reason to support WTF-16
rather than just UTF-16.

An important secondary reason is interoperability with JavaScript hosts.
For zero-copy interoperation with JavaScript and DOM facilities, it
would be good if for `stringref` to have the same semantics as a
JavaScript string, which like Java is an arbitrary sequence of 16-bit
code units.

Isolated surrogates are rare in JavaScript, but can occur via:
  - Reading invalid UTF-16 from external sources.  However this is not
    common, as most services prefer UTF-8 over UTF-16 as an interchange
    format.
  - JavaScript code that creates strings whose code units are not valid
    UTF-16.
  - JavaScript code that processes strings in chunks and happens to
    split a chunk on a surrogate boundary.
    - This happens most often in JavaScript code that processes strings
      one code unit at a time.
  - [JavaScript / DOM keyboard input event
    handlers](https://github.com/rustwasm/wasm-bindgen/issues/1348#issuecomment-473186426)
    (though this may be just a bug;
    [[1]](https://bugzilla.mozilla.org/show_bug.cgi?id=1541349),
    [[2]](https://bugs.chromium.org/p/chromium/issues/detail?id=949056)).

Therefore we define a `stringref` as an arbitrary sequence of not just
unicode scalar values, but also isolated surrogates.  Note that this
definition excludes codepoint sequences containing proper surrogate
pairs.  This restriction is enforced by construction for the WTF-8 and
WTF-16 encoding schemes.

### Are stringrefs mutable?

No.  We don't need mutable strings when compiling Java, C#, or Python,
and we don't need them when interoperating with JavaScript hosts.
Immutable strings have the benefit that you can hand them to an
untrusted interface without copying, and you know that interface won't
be able to use the string to affect any of your own state.

It is not a goal for `stringref` to be the main string representation
for programming languages that need mutable strings.  Fortunately there
are fewer and fewer of these languages as time goes on.

### What do existing C++ APIs to JavaScript strings look like?

While developing this proposal, we realized that we might already have a
design oracle as regards JavaScript integration: `v8.h`.  Perhaps for
languages that tend to work on strings in linear memory (C++, Rust), we
can use the C++ interface to a JS engine as an indication of what
interfaces we might need.
 - We can assume that `v8.h` has all the interfaces that Chromium needs,
   so we expect that the interfaces in `v8.h` are sufficient.
 - V8 wants to minimize API surface and historically has removed API, so
   we expect V8's interface is close to minimal.
 - C++ interfaces to different JS engines are similar.  We can look at
   `v8.h` and draw conclusions for any engine.

The [V8 C++ String
API](https://chromium.googlesource.com/v8/v8/+/refs/heads/main/include/v8-primitive.h#110)
includes the following procedural interfaces:

  - Create a string from encoded bytes in memory
    - Supported encodings: `one-byte`, `utf-8`, `utf-16`
  - Get length of string when encoded as `one-byte`, `utf-8`, `utf-16`
    - Does not include unicode scalar value count
  - Predicate on string to identify strings represented using one byte
    per character (a cheap check) and strings that can be represented
    using one byte per character (possibly a linear search)
  - Write encoded bytes to memory
    - Supported encodings: `one-byte`, `utf-8`, `utf-16`
    - Options: hint that ropes should be flattened, include `NUL`
      terminator or not, whether to preserve `NUL` codepoints, whether
      to replace isolated surrogates with the replacement character or
      to trap
  - Support for strings whose characters are in linear memory and which
    shouldn't be copied ("external strings"); probably not appropriate
    for WebAssembly
  - Equality predicate
  - Concatenate two strings.  Interestingly, `v8.h` has no interface to
    make a substring (slice).

We used this set of interfaces as a starting point for the `stringref`
design.  The need to support WebAssembly implementations that use WTF-8
to represent strings internally did cause us, however, to separate out
some functionality into `stringview`.

### What is the expected implementation on non-browser run-times?

Assuming that the non-browser implementation uses WTF-8 as the native
string representation, then a `stringref` could be just a pointer, a
length, and a reference count.  Some implementations may also want to
keep a flag indicating whether a string is valid UTF-8.

Generally speaking, WebAssembly doesn't specify the time or space
complexity of its operations.  In that regard, an implementation is free
to implement e.g. `string.concat` via an eager copy.  In practice
however we expect the same dynamics that lead JavaScript implementations
to natively support ropes and slices would hold with non-browser
run-times.  These implementations would also have their own heuristics
for when to flatten strings.

When creating a `stringview_wtf16` from a `stringref` on a system that
represents `stringref` as WTF-8, we expect that some implementations
will eagerly copy the string to a WTF-16 encoding.  Others will want to
implement a map from WTF-16 position to WTF-8 position via
[breadcrumbs](https://www.swift.org/blog/utf8-string/#breadcrumbs).

### What's the expected implementation in web browsers?

We expect that web browsers use JS strings as `stringref`.

We expect also that web browsers use JS strings directly as their
`stringview_wtf16` implementation, given that current web browsers
represent strings internally as WTF-16 (with some optimizations for
latin-1 strings).

For `stringview_wtf8`, we expect either an eager copy or breadcrumbs, as
in the non-browser runtime case.  Some small strings may avoid the
re-encoding and instead re-encode on the fly.

There is a possibility that some web browsers may eventually switch from
the one-byte/two-byte representation to WTF-8 with breadcrumbs, which
would make those web browsers use the same strategy as the non-browser
case.

### How should stringviews be represented in JavaScript hosts?

It's possible for a WebAssembly module to define an exported function
that returns a `stringview_iter`.  This proposal leaves the question of
the JS API for stringviews to a post-MVP proposal.  We expect that until
such a proposal lands, attempting to pass a stringview across the
WebAssembly/JS boundary will throw an exception, as was the case for
`i64` values before the BigInt proposal landed.

### How do we expect Rust to compile to `stringref`?

Generally speaking, for Rust we expect eager copies to UTF-8 data when
Rust receives a stringref.

Rust represents strings natively as well-formed UTF-8.  Rust string
processing routines can therefore assume that a UTF-8 string is valid.
`stringref` strings are WTF-8, though.  So we can expect that for a Rust
interface that exports a function that takes a `stringref` parameter,
`wasm-bindgen` would then use a
[WasmString](https://rustwasm.github.io/wasm-bindgen/reference/types/string.html)
type, which could be transformed to an `Option<String>` ([which could
fail or replace with U+FFFD if there are isolated
surrogates](https://rustwasm.github.io/wasm-bindgen/reference/types/str.html#utf-16-vs-utf-8)).
This will remove the need for `TextDecoder`/`TextEncoder`.

As an optimization for Rust modules that are designed to work with
WebAssembly, `WasmString` may expose some methods to avoid an eager
copy.

### How do we expect JVM and CLR languages to compile to `stringref`?

We expect Java to use `stringref` directly to represent string values.

Java deals with strings as immutable sequences of 16-bit code units.
Access to individual code units would use `stringview_wtf16`.

Alternately, a Java compiler might instead choose to use
`stringview_wtf16`, eagerly obtaining WTF-16 views when it receives a
`stringref` from the outside world.

### How do we expect Python to compile to `stringref`?

We expect CPython to provide a wrapper around `stringref` for strings
that come from "outside".  We expect PyPy to use `stringref` directly
for all strings.

Python strings [are immutable sequences of Unicode code
points](https://docs.python.org/3/library/stdtypes.html#textseq).  This
may include surrogates.

CPython's string support is abstract: all codepoint access goes through
an accessor API.  Therefore when CPython receives a `stringref` on a
public interface, CPython could store that `stringref` in a table and
then forward any indexed codepoint access to that `stringref`.

PyPy would instead use `stringref` directly to implement its strings.
The PyPy maintainer notes that most strings in Python aren't accessed
using indexed accessors, so probably PyPy would only obtain a view as
needed.

### How do we expect C++ to compile to `stringref`?

We expect that LLVM will be extended with an additional reference type,
`stringref`, like the existing `externref` and `funcref` support, along
with a number of builtins to expose the basic `stringref` operations.
LLVM will be able to directly expose C++ functions to WebAssembly that
take `stringref` parameters, removing the need for much Emscripten-side
code.  However as reference-typed values aren't storable to main memory,
we expect that unless a C++ program is carefully built to integrate
reference types, that most `stringref` values will be eagerly converted
to WTF-8 on the WebAssembly boundary.

### Is the `stringref` type nullable?

Oh God I guess so.  `ref.null string` it is I guess!!  :sob: :sob: :sob:

### What kinds of performance gains can we expect?

 * WebAssembly can receive encoded content of JS strings exactly where
   it is wanted: no need to stack-allocate then copy.
 * WebAssembly can process long strings in chunks rather than having to
   reserve space for the whole string.
 * WebAssembly can cheaply check incoming strings against literals,
   treating them as symbols.
 * Avoid JIT warmup for JS-implemented UTF-8 encode and decode.
 * Avoid allocation of subarrays when decoding; e.g. [as used by
   emscripten](https://github.com/emscripten-core/emscripten/blob/da842597941f425e92df0b902d3af53f1bcc2713/src/runtime_strings.js#L48)
 * Cheap prefix/suffix tests without reading whole string
 * WebAssembly can cheaply pass string literals to JS without decoding
   or copying

### What's the security implication?

Right now, working with strings fundamentally means communicating UTF-8
via memory.  To grant someone access to a string, you have to grant them
access to all of your memory.  This violates the principle of least
privilege.  Having reference-typed strings would limit the capability to
just the immutable codepoint sequence in question, and not all of
memory.

Additionaly, interfacing between memory lifetimes in C/C++ and
JavaScript is bug-prone.  Using `stringref` would eliminate questions of
memory ownership, reducing the risk of use-after-free, data corruption,
write overruns, and privileged data leakage.

### Why might you want to eagerly copy string contents to and from linear memory?

Some programming languages will be happy to deal with string contents
via the `stringview` APIs, avoiding copies of string contents to linear
or GC-managed memory.  Some others will prefer to copy out a WTF-8
encoding to main memory, because that's how they are used to dealing
with strings.  This copying has an overhead but for algorithms that
touch many code units it can be advantageous, as you get to inline the
per-code-unit processing work rather than calling out to `stringview`
interfaces.

As the `stringview` interfaces may exhibit polymorphism, they may have
some per-operation overheads.  For example, a `stringview_wtf16` will be
cheap to create in JavaScript, but accessing the code units still has to
dispatch over whether the string is a rope or a slice, whether the
codepoints are one-byte or two-byte, and so on.  Even in non-browser
WTF-8 implementations there will still be ropes and slices.

### Why not just use `externref` and imported functions?

The instruction set could be implemented with imported functions,
replacing the `stringref` type with `externref`.  So why bother adding
it to WebAssembly itself?  Three reasons: platform expressivity,
performance, and security.

On the first point: if the strings feature required some capability from
the host, then it would be clearly best as a library.  For example,
WebGL access falls in this category.  But reference-typed strings are a
more fundamental feature common to all languages that use automatic
memory management.  In that way they are closer to the GC proposal;
although you could implement structs and arrays via `externref` and
imports, if you did that you might as well compile to JavaScript instead
of WebAssembly.  It should be possible to make a WebAssembly program
that uses reference-typed strings (because almost all such programs
would have strings) without relying on any JavaScript at all.

Also, the evolutionary endpoint of an `externref`-and-imports strategy
is a JavaScript-specific string interface.  Without any broader
WebAssembly platform concern, strings-using WebAssembly code would find
itself relying on details of JavaScript's string representation, for
example having the only interface be to process strings one code unit at
a time instead of one codepoint at a time.  This is not a good platform
outcome.

Finally, though the WebAssembly platform should be able to stand alone,
it should also interoperate smoothly with hosts, especially JavaScript
on the web.  This rules out any implementation of strings in terms of
reference-typed structs and arrays: not only would such an
implementation be likely slower than the host's strings, it would also
be incompatible.  On the web, WebAssembly and JavaScript should use the
same string implementation.

On the performance side, we expect that `stringref` will be
faster than `externref`+imports:

 1. Whereas an `externref` might need to be a tagged union, a
    `stringref` can be an unpacked pointer.
 2. WebAssembly instructions are likely faster and less of an
    optimization barrier than callouts to imports.
 3. Run-time helper code for WebAssembly instructions is probably
    implemented in C++/Rust/etc more directly, resulting in more
    predictable performance than e.g. an encoder implemented in JS (for
    web embeddings).
 4. Reading string contents, either via
    `string.encode_wtf8`-then-process-inline or via `stringview_wtf16`,
    is likely faster than calling out to JavaScript to read code units
    one at a time.  WebAssembly-to-JavaScript calls are cheap but not
    free.

On the other hand, it's true that JS run-time routines can use adaptive
JIT techniques to possibly inline representation-specific accessors.
This is of limited use though for run-time routines with many different
call sites.

On the reliability and security side, adding `stringref` to WebAssembly
removes a significant user of extra-module access to memory.  Because
the WebAssembly code can pick apart the string itself, that's one fewer
reason for the WebAssembly module to have to expose its memory.

### How does this relate to the component model and interface types?

The component model is [a vision of how to compose systems out of
shared-nothing parts implemented in
WebAssembly](https://github.com/WebAssembly/component-model/pull/1/files).
The boundaries between these components are mediated by interface types,
which specify how to communicate data from one component to another in
the most efficient way possible.

At one level, reference-typed strings don't appear have anything to do
with the component model.  Because components are specified to not share
anything, even GC-managed data, zero-copy communication of
reference-typed strings between components is strictly out of scope
(though this operation may be zero-copy in practice; see below).

From the perspective of the component model, reference-typed strings are
rather an *intra-component* concern.  A component may be composed
internally of a number of WebAssembly modules, as well as possible host
facilities such as JavaScript.  The zero-copy properties provided by
`stringref` are only assured on to the inter-module, intra-component
boundaries of a program.

That said, strings in the abstract are an important data type, and
relate to interface types (a WebAssembly proposal based on the component
model).  Obviously you will want to be able to use `stringref` with
interface types.  The shared-nothing design choice of the component
model then implies that `stringref` contents should be copied when they
cross a component boundary.

Incidentally, for inter-component interfaces that deal in strings, the
component model specifies that abstractly, [strings are sequences of
unicode scalar
values](https://github.com/WebAssembly/interface-types/issues/135).
This implies that some JavaScript strings can't traverse a component
boundary, because of the potential for isolated surrogates, and also
implies an eager check that a `stringref` is a valid USV sequence, for
an interface-typed call.  In practice this is not a problem because the
`stringref` contents are being copied anyway and so can be validated at
the same time.

Interface types are used to specify a WebAssembly function's signature
in an abstract way.  This signature should then be compiled down to a
concrete adapter function specialized to the data representations used
by the caller and the callee.  The instruction set in this proposal can
be used to implement the adapter function for passing a `stringref` as a
string; assuming that the adapter function is generated in such a way
that it has access to the target memory, `string.encode_wtf8` can
implement the copy and validation at the same time.  `string.new_wtf8`
would be the implementation of getting a `stringref` from an
interface-typed string value, again assuming UTF-8 encoding for these
values.

Of course, because a `stringref` is immutable, whether it is copied or
not on a component boundary or during a call to an interface-typed
function is an implementation detail.  Some implementations of the
component model may wish to copy in all cases, for memory usage
accounting reasons.  Others will apply a zero-copy strategy when
possible, for example when both the caller and the callee of an
interface are implemented with `stringref`.  In the zero-copy case,
however, hosts have to eagerly verify that the string is a valid USV
sequence.  For this they would use `string.is_usv_sequence`.
