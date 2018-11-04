- Feature Name: binary_operator_symmetry
- Start Date: 2018-11-04
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

We propose adding the `.leading_ones()`, and `.trailing_ones()` methods to
numbers as counterparts to `.leading_zeros()`, and `.trailing_zeros()`.

# Motivation
[motivation]: #motivation

## Symmetry
In Rust there are currently 4 operations that count ones and zeros inside the
binary representation of numbers:

- `<num>::count_ones()`
- `<num>::count_zeros()`
- `<num>::leading_zeros()`
- `<num>::trailing_zeros()`

`count_zeros()` has a counterpart in `count_ones()`, but all other methods
do not have any counterparts.

In the Rust ecosystem it's easy to get used to the symmetry in methods. For
example: `Option::is_none()` is balanced with `Option::is_some()`, and
`<num>::rotate_left()` is balanced with `<num>::rotate_right()`.

Adding counterparts to `.leading_zeros()` and `.trailing_zeros()` would create
symmetry in those APIs, which seems like a natural fit for Rust's stdlib.

## Expressiveness & Reducing Errors
To count the number of trailing ones in a number, you'd currently have to write
something similar to:
```rust
fn do_something(i: u8) {
  let x = (!i).count_zeros();
  // (...)
}
```

This is not ideal for a few reasons. The binary negation operator (`!`) might
not be familiar for a lot of people. In languages such as JavaScript it would
evaluate to a boolean, and for people coming from those languages it might be a
little confusing at first sight what's going on.

Also there's some operator precedence to learn here: the negation operator is
evaluated after the method call. This requires the addition of braces to ensure
the statements are evaluated in the right order.

Instead this would be more expressive, and less prone to errors:

```rust
fn do_something(num: u8) {
  let x = i.count_ones();
  // (...)
}
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## `<num>::trailing_ones()`
The `.trailing_ones()` method would return the number of trailing ones in the
binary representation of the number.

```rust
let n = 0b1001011u8;
assert_eq!(n.trailing_ones(), 2);
````

This method would be available on all numerical types.

## `<num>::leading_ones()`
The `.leading_ones()` method would return the number of leading ones in the
binary representation of the number.

```rust
let n = 0b1001011u8;
assert_eq!(n.leading_ones(), 1);
````

This method would be available on all numerical types.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## `<num>::trailing_ones()`
The `.trailing_ones()` method would be implemented as part of
[`libcore/num/mod.rs`](https://github.com/rust-lang/rust/blob/f6e9a6e41cd9b1fb687e296b5a6d4c6ad399f862/src/libcore/num/mod.rs),
and added to each numerical type:

```rust
#[rustc_const_unstable(feature = "const_int_ops")]
#[inline]
pub const fn trailing_ones(self) -> u32 {
    (!self).trailing_zeros()
}
```

This implementation would be similar to
[`count_zeros()`](https://github.com/rust-lang/rust/blob/f6e9a6e41cd9b1fb687e296b5a6d4c6ad399f862/src/libcore/num/mod.rs#L306-L308).

The difference in (optimized) x86 assembly [would be a single
instruction](https://godbolt.org/z/iqBTBf):

```asm
trailing_zeros:
  movzx   eax, dil
  or      eax, 256
  tzcnt   eax, eax
  ret

trailing_ones:
  not     dil       ; added
  movzx   eax, dil
  or      eax, 256
  tzcnt   eax, eax
  ret
```

## `<num>::leading_ones()`
The `.leading_ones()` method would be implemented as part of
[`libcore/num/mod.rs`](https://github.com/rust-lang/rust/blob/f6e9a6e41cd9b1fb687e296b5a6d4c6ad399f862/src/libcore/num/mod.rs),
and added to each numerical type:

```rust
#[rustc_const_unstable(feature = "const_int_ops")]
#[inline]
pub const fn leading_ones(self) -> u32 {
    (!self).leading_zeros()
}
```

This implementation would be similar to
[`count_zeros()`](https://github.com/rust-lang/rust/blob/f6e9a6e41cd9b1fb687e296b5a6d4c6ad399f862/src/libcore/num/mod.rs#L306-L308).

The difference in (optimized) x86 assembly [would be a single
instruction](https://godbolt.org/z/L23G-0):

```asm
leading_zeros:
  movzx   eax, dil
  lzcnt   eax, eax
  add     eax, -24
  movzx   eax, al
  ret

leading_ones:
  not     dil       ; added
  movzx   eax, dil
  lzcnt   eax, eax
  add     eax, -24
  movzx   eax, al
  ret
```

# Drawbacks
[drawbacks]: #drawbacks

## API Surface Growth
Adding this feature would increase the overall API of Rust by introducing two
new methods to every number type. However this increase only adds symmetrical
counterparts to existing APIs, so there should be a minimal amount of extra
learning needed.

## No Direct Mapping To Intrinsics
The addition of these methods doesn't have a direct mapping to any Rust
intrinsics, and/or LLVM intrinsics.

For `.trailing_zeros` there's the
[`cttz intrinsic`](https://doc.rust-lang.org/std/intrinsics/fn.cttz.html), which
in LLVM maps onto
[`llvm.cttz`](https://llvm.org/docs/LangRef.html#llvm-cttz-intrinsic). A similar
mapping exists in `.leading_zeros()` through `ctlz`. This mapping would not
exist for the two APIs we're proposing in this RFC.

However there is prior art for a non-direct mapping in `.count_zeros()` and
`.count_ones()`.  `.count_zeros()` is implemented as [a negation of
`.count_ones()`](https://github.com/rust-lang/rust/blob/f6e9a6e41cd9b1fb687e296b5a6d4c6ad399f862/src/libcore/num/mod.rs#L306-L308),
which would be similar to what we're proposing:

```rust
pub const fn count_zeros(self) -> u32 {
  (!self).count_ones()
}
```

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Do Nothing
This RFC doesn't introduce anything that wasn't possible before, but instead
intends to codeify patterns into methods. This lowers the bar for accessing
this functionality, and provides a more streamlined experience.

## Introduce As A `NumExt` library
It might be argued that convenience methods should live in user land, as part of
libraries that provide extension traits to stdlib.

However, that would seem like it would have the same discovery problems as
learning the `(!<num>).<method>` pattern, and could arguably be considered too
small of a library which probably would see little usage. Making it effectively
the same as doing nothing.

## Naming
Arguably using the `*_ones()` suffix might lead to confusing. When reading
`.trailing_ones()` in code without context, it might be reasonable to ask:
"Which ones are trailing?", and "Do they feel lonely?".

An alternative naming scheme might include a `*_1s()` suffix, removing some of
the ambiguity. However this would break the precedent set in methods such as
`.count_ones()`. Therefor it seems best to follow the existing scheme of pairing
`zeros` with `ones`.

# Prior art
[prior-art]: #prior-art

## Rust's Stdlib
Rust's `<num>::count_zeros()` method is implemented as a binary negation of
`<num>::count_ones()`, which maps to the `popcnt` instruction.

In x86 assembly, the [difference between the two
methods](https://godbolt.org/z/ArBv8P) is a single `not` instruction:

```asm
count_ones:
  movzx   eax, dil
  popcnt  eax, eax
  ret
```

```asm
count_zeros:
  not     dil        ; added
  movzx   eax, dil
  popcnt  eax, eax
  ret
```


For the methods we're proposing, [the difference would be
similar](#guide-level-explanation).

## C++
We're currently not aware of any prior art for this method in other languages.
Most of the prior art we've found has been through access to compiler intrinsics
such as [gcc's
`__builtin_clz`](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html) and
[msvc's `popcnt`](https://msdn.microsoft.com/en-us/library/bb385231.aspx).

A quick search on the internet brings up examples to manually combine these
methods such as:

```cpp
#ifdef __GNUG__
int count_ones (unsigned int a) {
  return __builtin_popcount(a) ;
}
#endif // __GNUG__

#ifdef _MSC_VER
int count_ones (unsigned int a) {
  return __popcount(a);
}
#endif // _MSC_VER
```

This seems to have a reasonable amount of boilerplate, and is less convenient
than having a single, portable method available.

## Dat Protocol
The Dat protocol makes use of a data structure called ["Flat in-order
trees"](https://github.com/datprotocol/DEPs/blob/master/proposals/0002-hypercore.md#flat-in-order-trees),
where entries have a "depth" property that can be calculated by counting the
amount of trailing ones in the binary representation of numbers.

```txt
 0─┐
   1─┐
 2─┘ │
     3
 4─┐ │
   5─┘
 6─┘

5 in binary = 101 (one trailing 1)
3 in binary = 011 (two trailing 1s)
4 in binary = 100 (zero trailing 1s)
```

It seems reasonable to assume that more algorithms exist that make use of
similar number properties.

# Unresolved questions
[unresolved-questions]: #unresolved-questions
None.

# Future possibilities
[future-possibilities]: #future-possibilities

While researching how to most efficiently count trailing ones, we found some
interesting operations that currently don't seem to be exposed in Rust.

For example Intel's
[`bit_scan_*`](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#text=bit_scan&expand=392,393,400,401)
range of instructions might be interesting to add.

There's probably a balance to be had, as there's probably such a thing as "too
many methods available". But it might still be interesting for Rust to look at
which other methods could be exposed, as it could enable some interesting
optimizations for algorithms that might otherwise require some guesswork to
achieve.

This would be similar to how `SIMD` intrinsics have recently be introduced, as a
no-guesswork counterpart to auto-vectorization.
