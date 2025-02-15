---
section: getting-started
layout: getting-started
title: Binaries, strings, and charlists
---

In "Basic types", we learned a little bit about strings and we used the `is_binary/1` function for checks:

```elixir
iex> string = "hello"
"hello"
iex> is_binary(string)
true
```

In this chapter, we will gain clarity on what exactly binaries are, how they relate to strings, and what single-quoted values, `'like this'`, mean in Elixir. Although strings are one of the most common data types in computer languages, they are subtly complex and are often misunderstood. To understand strings in Elixir, we have to educate ourselves about [Unicode](https://en.wikipedia.org/wiki/Unicode) and character encodings, specifically the [UTF-8](https://en.wikipedia.org/wiki/UTF-8) encoding.

## Unicode and Code Points

In order to facilitate meaningful communication between computers across multiple languages, a standard is required so that the ones and zeros on one machine mean the same thing when they are transmitted to another. The [Unicode Standard](https://unicode.org/standard/standard.html) acts as an official registry of virtually all the characters we know: this includes characters from classical and historical texts, emoji, and formatting and control characters as well.

Unicode organizes all of the characters in its repertoire into code charts, and each character is given a unique numerical index. This numerical index is known as a [Code Point](https://en.wikipedia.org/wiki/Code_point).

In Elixir you can use a `?` in front of a character literal to reveal its code point:

```elixir
iex> ?a
97
iex> ?ł
322
```

Note that most Unicode code charts will refer to a code point by its hexadecimal representation, e.g. `97` translates to `0061` in hex, and we can represent any Unicode character in an Elixir string by using the `\u` notation and the hex representation of its code point number:

```elixir
iex> "\u0061" === "a"
true
iex> 0x0061 = 97 = ?a
97
```

The hex representation will also help you look up information about a code point, e.g. [https://codepoints.net/U+0061](https://codepoints.net/U+0061) has a data sheet all about the lower case `a`, a.k.a. code point 97.

## UTF-8 and Encodings

Now that we understand what the Unicode standard is and what code points are, we can finally talk about encodings. Whereas the code point is **what** we store, an encoding deals with **how** we store it: encoding is an implementation. In other words, we need a mechanism to convert the code point numbers into bytes so they can be stored in memory, written to disk, etc.

Elixir uses UTF-8 to encode its strings, which means that code points are encoded as a series of 8-bit bytes. UTF-8 is a **variable width** character encoding that uses one to four bytes to store each code point; it is capable of encoding all valid Unicode code points.

Besides defining characters, UTF-8 also provides a notion of graphemes. Graphemes may consist of multiple characters that are often perceived as one. For example, `é` can be represented in Unicode as a single character. It can also be represented as the combination of the character `e` and the acute accent character `´` into a single grapheme.

In other words, what we would expect to be a single character, such as é, can in practice be multiple codepoints (in this case, e and an acute accent), each represented by potentially multiple bytes. Consider the following:

```elixir
iex> string = "héllo"
"héllo"
iex> String.length(string)
5
iex> length(String.to_charlist(string))
6
iex> byte_size(string)
7
```

`String.length/1` counts graphemes and returned 5. To count the number of code points, we can use `String.to_charlist/1` to convert a string to a list of codepoints, and then we get its length, which returned 6. Finally, `byte_size/1` reveals the number of underlying raw bytes needed to store the string when using UTF-8 encoding. UTF-8 requires one byte to represent the characters `h`, `e`, `l`, and `o`, but two bytes to represent the acute accent, adding to 7.

> Note: if you are running on Windows, there is a chance your terminal does not use UTF-8 by default. You can change the encoding of your current session by running `chcp 65001` before entering `iex` (`iex.bat`).

A common trick in Elixir when you want to see the inner binary representation of a string is to concatenate the null byte `<<0>>` to it:

```elixir
iex> "hełło" <> <<0>>
<<104, 101, 197, 130, 197, 130, 111, 0>>
```

Alternatively, you can view a string's binary representation by using [IO.inspect/2](https://hexdocs.pm/elixir/IO.html#inspect/2):

```elixir
iex> IO.inspect("hełło", binaries: :as_binaries)
<<104, 101, 197, 130, 197, 130, 111>>
```

We are getting a little bit ahead of ourselves. Let's talk about bitstrings to learn about what exactly the `<<>>` constructor means.

## Bitstrings

Although we have covered code points and UTF-8 encoding, we still need to go a bit deeper into how exactly we store the encoded bytes, and this is where we introduce the **bitstring**. A bitstring is a fundamental data type in Elixir, denoted with the `<<>>` syntax. **A bitstring is a contiguous sequence of bits in memory.**

A complete reference about the binary / bitstring constructor `<<>>` can be found [in the Elixir documentation](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#%3C%3C%3E%3E/1).

By default, 8 bits (i.e. 1 byte) is used to store each number in a bitstring, but you can manually specify the number of bits via a `::n` modifier to denote the size in `n` bits, or you can use the more verbose declaration `::size(n)`:

```elixir
iex> <<42>> === <<42::8>>
true
iex> <<3::4>>
<<3::size(4)>>
```

For example, the decimal number `3` when represented with 4 bits in base 2 would be `0011`, which is equivalent to the values `0`, `0`, `1`, `1`, each stored using 1 bit:

```elixir
iex> <<0::1, 0::1, 1::1, 1::1>> == <<3::4>>
true
```

Any value that exceeds what can be stored by the number of bits provisioned is truncated:

```elixir
iex> <<1>> === <<257>>
true
```

Here, 257 in base 2 would be represented as `100000001`, but since we have reserved only 8 bits for its representation (by default), the left-most bit is ignored and the value becomes truncated to `00000001`, or simply `1` in decimal.

## Binaries

**A binary is a bitstring where the number of bits is divisible by 8.** That means that every binary is a bitstring, but not every bitstring is a binary. We can use the `is_bitstring/1` and `is_binary/1` functions to demonstrate this.

```elixir
iex> is_bitstring(<<3::4>>)
true
iex> is_binary(<<3::4>>)
false
iex> is_bitstring(<<0, 255, 42>>)
true
iex> is_binary(<<0, 255, 42>>)
true
iex> is_binary(<<42::16>>)
true
```

We can pattern match on binaries / bitstrings:

```elixir
iex> <<0, 1, x>> = <<0, 1, 2>>
<<0, 1, 2>>
iex> x
2
iex> <<0, 1, x>> = <<0, 1, 2, 3>>
** (MatchError) no match of right hand side value: <<0, 1, 2, 3>>
```

Note that unless you explicitly use `::` modifiers, each entry in the binary pattern is expected to match a single byte (exactly 8 bits). If we want to match on a binary of unknown size, we can use the `binary` modifier at the end of the pattern:

```elixir
iex> <<0, 1, x::binary>> = <<0, 1, 2, 3>>
<<0, 1, 2, 3>>
iex> x
<<2, 3>>
```

There are a couple other modifiers that can be useful when doing pattern matches on binaries. The `binary-size(n)` modifier will match `n` bytes in a binary:

```elixir
iex> <<head::binary-size(2), rest::binary>> = <<0, 1, 2, 3>>
<<0, 1, 2, 3>>
iex> head
<<0, 1>>
iex> rest
<<2, 3>>
```

**A string is a UTF-8 encoded binary**, where the code point for each character is encoded using 1 to 4 bytes. Thus every string is a binary, but due to the UTF-8 standard encoding rules, not every binary is a valid string.

```elixir
iex> is_binary("hello")
true
iex> is_binary(<<239, 191, 19>>)
true
iex> String.valid?(<<239, 191, 19>>)
false
```

The string concatenation operator `<>` is actually a binary concatenation operator:

```elixir
iex> "a" <> "ha"
"aha"
iex> <<0, 1>> <> <<2, 3>>
<<0, 1, 2, 3>>
```

Given that strings are binaries, we can also pattern match on strings:

```elixir
iex> <<head, rest::binary>> = "banana"
"banana"
iex> head == ?b
true
iex> rest
"anana"
```

However, remember that binary pattern matching works on *bytes*, so matching on the string like "über" with multibyte characters won't match on the _character_, it will match on the _first byte of that character_:

```elixir
iex> "ü" <> <<0>>
<<195, 188, 0>>
iex> <<x, rest::binary>> = "über"
"über"
iex> x == ?ü
false
iex> rest
<<188, 98, 101, 114>>
```

Above, `x` matched on only the first byte of the multibyte `ü` character.

Therefore, when pattern matching on strings, it is important to use the `utf8` modifier:

```elixir
iex> <<x::utf8, rest::binary>> = "über"
"über"
iex> x == ?ü
true
iex> rest
"ber"
```

You will see that Elixir has excellent support for working with strings. It also supports many of the Unicode operations. In fact, Elixir passes all the tests showcased in the article ["The string type is broken"](http://mortoray.com/2013/11/27/the-string-type-is-broken/).

## Charlists

Our tour of our bitstrings, binaries, and strings is nearly complete, but we have one more data type to explain: the charlist.

**A charlist is a list of integers where all the integers are valid code points.** In practice, you will not come across them often, except perhaps when interfacing with Erlang, in particular when using older libraries that do not accept binaries as arguments.

Whereas strings (i.e. binaries) are created using double-quotes, charlists are created with single-quoted literals:

```elixir
iex> 'hello'
'hello'
iex> [?h, ?e, ?l, ?l, ?o]
'hello'
```

You can see that instead of containing bytes, a charlist contains integer code points. However, the list is only printed in single-quotes if all code points are within the ASCII range:

```elixir
iex> 'hełło'
[104, 101, 322, 322, 111]
iex> is_list('hełło')
true
```

Interpreting integers as code points may lead to some surprising behavior. For example, if you are storing a list of integers that happen to range between 0 and 127, by default IEx will interpret this as a charlist and it will display the corresponding ASCII characters.

```elixir
iex> heartbeats_per_minute = [99, 97, 116]
'cat'
```

You can convert a charlist to a string and back by using the `to_string/1` and `to_charlist/1` functions:

```elixir
iex> to_charlist("hełło")
[104, 101, 322, 322, 111]
iex> to_string('hełło')
"hełło"
iex> to_string(:hello)
"hello"
iex> to_string(1)
"1"
```

Note that those functions are polymorphic - not only do they convert charlists to strings, they also operate on integers, atoms, and so on.

String (binary) concatenation uses the `<>` operator but charlists, being lists, use the list concatenation operator `++`:

```elixir
iex> 'this ' <> 'fails'
** (ArgumentError) expected binary argument in <> operator but got: 'this '
    (elixir) lib/kernel.ex:1821: Kernel.wrap_concatenation/3
    (elixir) lib/kernel.ex:1808: Kernel.extract_concatenations/2
    (elixir) expanding macro: Kernel.<>/2
    iex:1: (file)
iex> 'this ' ++ 'works'
'this works'
iex> "he" ++ "llo"
** (ArgumentError) argument error
    :erlang.++("he", "llo")
iex> "he" <> "llo"
"hello"
```

With binaries, strings, and charlists out of the way, it is time to talk about key-value data structures.
