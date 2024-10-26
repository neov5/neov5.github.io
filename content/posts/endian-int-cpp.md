---
title: "endianint.hpp: an endianness-maintaining templated integer type for C++20"
date: 2024-10-25T22:37:16+02:00
draft: true
---

## Motivation

Oasis' [virtio spec][1], Section 1.4 defines the endian-specific types `le16`,
`le32`, `le64` to be little-endian unsigned integers. Likewise for `be16`, `be32`
and `be64`. I was perusing this because I wanted to make my own userspace
networking drivers which I could run on a virtual machine. Practically every
machine you could choose to run a VM on is little endian, and you would have to 
[go out of your way to test that your code runs on big endian machines][2].

A puzzle is a puzzle, though. And I was doing this for fun, so let's have some 
fun.

## Requirements

We want to create a templated type which would take endianness and size of the 
integer as it's parameters. The main requirements of this type are:

* **zero-overhead**: the compiler should be capable of optimizing out 
  the class and it's methods as needed.
* **equivalence**: any C code interfacing with it should not be able to tell 
  the difference between our class and an integer. This (among other things) 
  implies they both take up 2 bytes
* **transparency**: I should be able to do arithmetic on the type as if it is an 
  integer in C++, without worrying about casting and other issues. If it is 
  operated on with native types, it's value should be cast into the native
  system's endianness transparently.

## Implementation

Here's the 16-bit implementation. The 32 and 64-bit implementations are very 
similar but have a longer byteswap, so I've omitted them for clarity.

```c++
#include <cstdint>
#include <type_traits>
#include <bit>

using namespace std;

template <endian E, typename T, typename Enable = std::enable_if_t<std::is_integral_v<T>>> class endianint;

static_assert(endian::native == endian::little or endian::native == endian::big,
              "Mixed endian machines are not supported");

template <endian E>
class endianint<E, uint16_t> {

    static constexpr uint16_t to_E(uint16_t val) {
        if constexpr ((E == endian::big and endian::native == endian::little) or 
                      (E == endian::little and endian::native == endian::big)) {
            return ((val & 0xFF00) >> 8u) |
                   ((val & 0x00FF) << 8u);
        }
        return val;
    }

    static constexpr uint16_t from_E(uint16_t val) {
        return to_E(val); // commutative op!
    }

public:
    uint16_t value;
    endianint() : value(0) {}
    endianint(uint16_t val) : value(to_E(val)) {}

    endianint& operator=(uint16_t val) {
        value = to_E(val);
        return *this;
    }

    operator uint16_t() const {
        return from_E(value);
    }

};
```

Some things immediately stand out:

* **`<bit>` is a C++20 feature**: this header enables the endian enum, and the 
  native endian checks. My guess is it would be possible to package this up 
  and ship it with the header for non-C++20 platforms, but I haven't looked 
  into it yet.
* **`constexpr` everywhere for zero-overhead**
* **a single member for equivalence**: these two are self-explanatory. We want 
  the compiler to do the heavy lifting, and these operations should take at most 
  one instruction in code
* **`=` and casting overriden for transparency**: overriding more operators would 
  not really make a difference. For example, if we do the following:
```c++
using le16 = endianint<endian::little, uint16_t>;
using be16 = endianint<endian::big, uint16_t>;

le16 n1(1);
be16 n2(2);
le16 n3;

n3 = n2 + n1;
```
  the addition would be equivalent to doing 
```c++
n3 = le16(uint16_t(n2) + uint16_t(n1));
```
  If we override `operator+` to add numbers with opposing endianness, we would 
  still need to do atleast one byteswap. In this case, the compiler would 
  firstly optimize out the casts and would secondly do the byteswap itself 
* **manual byteswap?**: C++23 introduced `std::byteswap`, which would use a 
  system-specific implementation (on x86 it's the `bswap` instruction). Most 
  compilers also have intrinsics for this, with `__builtin_bswap16` working 
  on both clang and gcc. However, the easiest method is to *let the compiler 
  optimize out the swap itself*. Most compilers are smart enough to recognize 
  that it's a byteswap, and will optimize it away into a `bswap` by themselves
  without any coaxing.

## Results



[1]: https://docs.oasis-open.org/virtio/virtio/v1.3/virtio-v1.3.html
[2]: https://waterjuiceweb.wordpress.com/2017/12/02/setting-up-a-virtual-big-endian-environment/
