# Bit Flag `Enum`

A bit flag `enum` consists of two types: a flag `enum` and a newtype which keeps track of which flags are set. (Example loosely derived from Maude.)

## Flag `enum`

Create an enum for the individual flags using a representation type with an appropriate number of bits. Assign them values that are powers of 2. This is simple to do in hexadecimal: start with $2^0=1$ and multiply by 2 to get the next number, keeping in mind that 0x2 $\times$ 0x8 = 0x10.

```rust
// Derive to taste.
#[derive(Copy, Clone, PartialEq, Eq, PartialOrd, Ord, Debug)]
#[repr(u16)]
enum NodeFlag {
  Prec         = 0x1,
  Gather       = 0x2,
  Format       = 0x4,
  Latex        = 0x8,
  Strat        = 0x10,
  Memo         = 0x20,
  Frozen       = 0x40,
  Ctor         = 0x80,
  Config       = 0x100,
  Object       = 0x200,
  Message      = 0x400,
  MsgStatement = 0x800,
  Assoc        = 0x1000,
  Comm         = 0x2000,
  Idem         = 0x4000,
  Iter         = 0x8000
}
```

## Newtype Flags "Container"

The "container" that holds the flags is a newtype for the `repr` type of the flag `enum`.

```rust
#[derive(Copy, Clone, PartialEq, Eq, PartialOrd, Ord, Debug)]
pub struct NodeFlags(u16);
```

We need a way to combine flag variants together and to set flags in the flag container. You could implement a `set(&mut self, flag: NodeFlag)` method on `NodeFlags`. The traditional way of combining flags, though, is with bitwise OR. We implement bit OR in the relevant combinations.

```rust
impl std::ops::BitOr<NodeFlag> for NodeFlags {
  type Output = Self;

  fn bitor(self, rhs: NodeFlag) -> Self::Output {
    NodeFlags(self.0 | rhs as u32)
  }
}

impl std::ops::BitOr for NodeFlags {
  type Output = Self;

  fn bitor(self, rhs: Self) -> Self::Output {
    NodeFlags(self.0 | rhs.0)
  }
}

impl From<NodeFlag> for NodeFlags {
  fn from(value: NodeFlag) -> Self {
    NodeFlags(value.0 as u32)
  }
}

impl std::ops::BitOr for NodeFlag {
  type Output = NodeFlags;

  fn bitor(self, rhs: Self) -> Self::Output {
    NodeFlags(self as u32 | rhs as u32)
  }
}

impl std::ops::BitOr<NodeFlags> for NodeFlag {
  type Output = NodeFlags;

  fn bitor(self, rhs: NodeFlags) -> Self::Output {
    NodeFlags(self as u32 | rhs.0)
  }
}
```

You can also optionally implement bitwise AND operations. We take a more economical solution and implement an `is_set(…)` method on the container (and optionally also on the flag enum, which would just check equality).

```rust
impl NodeFlags {
  pub fn is_set(&self, flag: NodeFlag) -> bool {
    return (flag as u16 & self.0) != 0;
  }
}
```

Be careful of the semantics of bitwise AND! It might be tempting to have a  `contains(…)` method take a `NodeFlags` instead of a `NodeFlag`:

```rust
impl NodeFlags {
  /// Is any single one of the flags in `flags` also set in `self`?
  pub fn contains(&self, flags: NodeFlags) -> bool {
    return (flags.0 & self.0) != 0;
  }
}
```

The semantics of this version of `contains` is, return true if and only if any single one of the flags in `flags` is also set in `self`. This is different from the semantics, return true if and only if *all* of the flags in `flags` are set in `self`. For the latter, use this:

```rust
impl NodeFlags {
  /// Are all of the flags in `flags` also set in `self`?
  pub fn contains(&self, flags: NodeFlags) -> bool {
    return (flags.0 & self.0) == flags.0;
  }
}
```

## Compound Flags

You can put named _compound_ flags, that is, flags that are the sum of other flags, in the `enum` if you want, but you will have to compute the numeric value yourself. The compiler will warn you in incomplete `match` expressions when you match on the `enum` type, which might or might not be what you want. Usually, I think of compound flags as distinct kinds of things from the individual flag variants, so instead I prefer to put them within `impl NodeFlag`, which gives them the same syntax as `enum` variants but without variant semantics. Capitalization visually indicates that they are compound. Note that they are of type `NodeFlags`. You could, of course, put them in `impl NodeFlags` instead, or whereever else makes the most sense to you.

```rust
// Compound Flags.
impl NodeFlag {
  const AXIOMS: NodeFlags =
    NodeFlag::Assoc
  | NodeFlag::Comm
  | NodeFlag::LeftId
  | NodeFlag::RightId
  | NodeFlag::Idem;

  const COLLAPSE: NodeFlags = NodeFlag::LeftId | NodeFlag::RightId | NodeFlag::Idem;

  const SIMPLE_ATTRIBUTES: NodeFlags =
    NodeFlag::Assoc
  | NodeFlag::Comm
  | NodeFlag::Idem
  | NodeFlag::Memo
  | NodeFlag::Ctor
  | NodeFlag::Config
  | NodeFlag::Object
  | NodeFlag::Message
  | NodeFlag::Iter

  const ATTRIBUTES: NodeFlags =
    NodeFlag::Prec
  | NodeFlag::Gather
  | NodeFlag::Format
  | NodeFlag::Latex
  | NodeFlag::Strat
  | NodeFlag::Memo
  | NodeFlag::Frozen
  | NodeFlag::Config
  | NodeFlag::Object
  | NodeFlag::Message
  | NodeFlags::AXIOMS
  | NodeFlag::Iter
}
```

## Alternative Crates

Several crates provide similar functionality. 
