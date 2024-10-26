---
title: "endianint.hpp: endianness-maintaining integers in 100 lines of C++(20)"
date: 2024-10-25T22:37:16+02:00
---

## Motivation

Oasis' [virtio spec][1], Section 1.4 defines the endian-specific types `le16`,
`le32`, `le64` to be little-endian unsigned integers. Likewise for `be16`, `be32`
and `be64`. I was perusing this because I wanted to make my own userspace
networking drivers which I could run on a virtual machine. Practically every
machine you could choose to run a VM on today is little endian, and you would have to 
[go out of your way to test that your code runs on big endian machines][2]. Ergo,
it would have been good enough to do `using le16 = uint16_t` and move on.

A challenge is a challenge though. And I was doing this for fun, so let's have some 
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

template <endian E, typename T, 
          typename Enable = std::enable_if_t<std::is_integral_v<T>>> class endianint;

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
* **`constexpr` everywhere for zero-overhead**: We want 
  the compiler to do the heavy lifting, and these operations should take at most 
  one instruction in code
* **a single member for equivalence**: storing the endianness or any other 
  parameters would take up more space than the underlying type
* **`=` and casting overriden for transparency**: overriding more operators would 
  not make a difference, as the *endianness of the underlying system determines 
  the endianness you can do arithmetic operations in*. For example, if we do 
  the following on a little endian system:

  ```c++
  using le16 = endianint<endian::little, uint16_t>;
  using be16 = endianint<endian::big, uint16_t>;

  le16 n1(1);
  be16 n2(2);
  be16 n3;

  n3 = n2 + n1;
  // equivalent to n3 = be16(uint16_t(n2) + uint16_t(n1))
  ```

  we would need to do two byteswaps regardless, as we can't add big-endian 
  numbers on a little endian machine. The sensible thing to do is to 
  convert to the system endianness whenever we perform any arithmetic 
  operations. If there are any tricks to do big-endian addition ie shift carries
  to the preceding byte rather than the following, I am not aware. Maybe some 
  combination of sse operations could do that, but I claim it would be slower 
  than doing the byteswap, the operation and then swapping back.
* **manual byteswap?**: C++23 introduced `std::byteswap`, which would use a 
  system-specific implementation (on x86 it's the `bswap` instruction). Most 
  compilers also have intrinsics for this, with `__builtin_bswap16` working 
  on both clang and gcc. However, the easiest method is to *let the compiler 
  optimize out the swap itself*. Most compilers are smart enough to recognize 
  that it's a byteswap, and will optimize it away into a `bswap` by themselves
  without any coaxing.

## Results

Let's first test out transparency: **in a little-endian system, code generated
using `le16` should be identical to code generated when using `uint16_t`**

<iframe width="800px" height="400px" src="https://godbolt.org/e?readOnly=true&hideEditorToolbars=true#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGe1wAyeAyYAHI%2BAEaYxCAArNIADqgKhE4MHt6%2BekkpjgJBIeEsUTHxtpj2eQxCBEzEBBk%2BflzllWk1dQQFYZHRcdIKtfWNWS2Dnd1FJf0AlLaoXsTI7BzmAMzByN5YANQma27Ig/iC%2B9gmGgCC65vbmHsHBACeCZgA%2BgTETIQKZxfXZg2DC2Xl2%2BzcEUIfyu/y8KSMO2YbAUCSYyx2x32VhhVwImBYCQMeIeblceEMO2wpB2z1eSPuABVqbTGKx7thmBF6A8ACIYgjoEAgVlc954KgfcHHIV4BRvYJ44BfWhvABu4IZfzW521Oy2TAUCh2ZMMCqx/3%2B40cyDeBqU9QgJr8IGYjlV932PM9xoY%2BEMQtohAI3JIPr9ztdeHdvO9TqFkOApH%2BOxTqbTafMZgAsnhVJh0GHyQwdiw0QhgpgjXV7gxUAQMV4Ekl6vnMzNzTjLniCUSPQcnZToZd9YbC6bTgcqTsvAquAA2SW6kwAdmx1yuaateGQeoEg0wqgSxGns4X9aIb2wEBngnnHx2qrEMz2q%2BT6ZT4t3DH3h%2BPUGwMZrHycYgAmOyGAWIGRtGnqxr6RYBkG9DPqGb7vuh6EQABsFAWOzqBgQwb3BBeFCtBfZerhIEJjMz4rmuGFpsQmAEIsxZQI%2BtB7GYc47BoqgAGICRoGh0dqZw7AAHF4dHLm4aGMZhECcdxvH8SJQlifJBxSTJHaXBhK5ehu6HMaxxDFpx%2BkZsuxnrgZqZbjuaDfniv4nreZ47FQxCoCwl7Xqe96cbJDGmSxbE0qgAUhViOwAPTxbuLAsF4tRuvcqAJGAYAKUZFpXAkXhctuIAKTeBB3vWnFeJg1mpk6CoQM%2BIAPmItUQKJL5WLZCmNYIgWecFT47K1NWYBAF5XiFoX5Z2ab9QQ5i8Vl0RMEQxCeoNlVeSF3UKWm428lFMVPvV75mZFABUBDlr8axhSmc32Wmq1fBtHk7R8zVfoM%2B0mRdEUWd5vn%2BdN7WYO2D15b1nb5dDnZwsEwA7PQ87HYt4IgQRRHUhVVVamuSMIl46PHfjZ76cTKNROjcHhma/bwf6oF4Imn0E%2BJCP2WjvFMOg6BvLzEC84iLSo5g6MMGYs0Azsl3AwwXDcRYiJmPpz2wuj/OC6Tc7XlL4t62rssOSmCvFkrKtqxrtkcHMtCcLEvB%2BNwvCoJw8mWNYGILEsHqAjwpAEJo9tzAA1nEZgAHRzsuazLnOkhzmYACcCdcDx%2BicJIvAsHEazRxoc6SZnZjLmYkipxokmp6nc6kK7WikB7HC8AoIAaMHodzHAsBIGgBJ0NE5CUIPCTDzEwBSGYfB0HixAdxAESh6QkLMMQTycEH691E8ADyETaJgDjb7wg9sII%2B8MLQW8cM3WARF4wBuGItAd27pBYKWRjiPfvD4GYg4KMlZV4HhPmlFYzcFQVFXoGCIXxN4eCwKvT4eB86f3dMQCIyRMA8nxIYYAgYjA9z4AYYACgABqeBMAAHd950jPjIQQIgxDsCkMw%2BQSg1Cr10C0AwJDTDe0sPoPAEQO6QDmFlKoH8AC0mIgJCKsJYMwGgdiyP3kHVAWDiB4CwBI5qrQT5VBcL6EYzRSCBArFMPoLQcipAEOY7IyQHEMEmL0GIYwKjGPaEMBongmh6DsD4gQHR6juOKLY2wfinFjD8RE6YXA5gKD9ssCQDsnYu1Xq3HYqhJJzlkUnHYwBkA7ikNHMwOwsL4A%2BusJJvAQ7/1oqQBAmB%2BZ9EMY7DgudSD52TtHdOXBJLLgrsuWIGhU5cA0C0Ju7tODt07t3JpEcQB1wGXXDZmyNnLmzhwNYWT/4t3mUsrQzSsEpGcJIIAA%3D"></iframe>

And it is, even with `-O1`! Let's see if we can add two little-endian points (a 
ordered pair of (x, y) le16's) together

<iframe width="800px" height="600px" src="https://godbolt.org/e?readOnly=true&hideEditorToolbars=true#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIAGykrgAyeAyYAHI%2BAEaYxAEapAAOqAqETgwe3r4ByanpAqHhUSyx8f6JdpgOGUIETMQEWT5%2BgVU1AnUNBEWRMXEJtvWNzTltwz1hfaUDFQCUtqhexMjsHOYAzGHI3lgA1CYbbsgKBPiCh9gmGgCCm9u7mAdHBACeSZgA%2BgTETIQKl2udzMWwYOy8%2B0ObmihEBtyBXjSRj2zDYCiSTFWe1O6EOVnhtwImBYSQMROeblceEMe2wpD2bw%2BqKeABV6YzGKwnthmNF6M8ACLYs4gECcvlfPBUb5QnGivAKT5hInAX60T4ANyhLMBGyuer2OyYCgUeyphmVeKBQNOTEcyE%2BxqUjQg5r8IGYjg1T0OAt9ZoY%2BEMotohAI/JIAaD7s9eG9gv9btFMOApCBewzmazWfMZgAsnhVJh0FHqQw9ixMQgpqaGk8GKgCNivEkUo1i7m5laCTciSSyT6jm7aXCbkaTaWLRcjnS9l5lVx/DKDSYAOz4u63LO2%2B2GgSnTCqJLEOcLpdNoifbAQeeCRffPYasRzA7r9PZjNSvcMA9Hk9QbAEw2IUkxAFM9kMEtQNjeNfUTQMyxDMN6BfSN3w/DCMIgQC4OAyd3VDAhwyeSD8NFGDBz9PDQJTOYXzXDdMKzYhMAIZZyygJ9aAOMx/D2DRVAAMUEjQNHovVLj2AAOLx6NXNx0KYrCIC4ni%2BIE0ThPEhSjmk2TuxuTC1z9LcMJYtjiHLLiDJzVcTM3QzMx3PBkG/X9j1PO9zz2KhiFQFgrxvM8Hy4uTGLM1j2IZVBAtCvE9gAegSvcWBYLx6i9J5UCSMAwEU4zrVuJIvD5FyQEU28CHvJsuK8TAbMzN1lQgF8QEfMQ6ogMTXysOzFKawQgq8kLnz2NraswCBL2vUKwoKnsswGghzD47K4jtEhfSGqrvNCnrFKzCbBWi2Lnwaj9zKigAqAhqwBDZwozeaHKzNbfiIE9Kuqlq3KbBiDszS7LJ8vyApmjrMC7B78r6nsCuhntETCYA9noRdjqWqFQMI4j6S%2B89dQ3JHkS8dHjvx5ciaRFHYnR%2BDo0tIcEODMC8FTTyduXfUNxtH4vAcPYkgIT40b4/7TIzUW9lUelXgM%2BGeduU5iH5pshc%2BUmxbfCW53RmW9jlhG7j6o2gXVqWmHQdBPnNzBFwgW30YYLh6UdviGDMOadaB8sGJRLgADpVB4iwUTMIP6WdgPXhDsPo9fKiedhhz1c1iCrZt4XNYdrOnZdwXc/dz39u9yLgb9qPg%2BsOP9ajmPq49%2BOFetOyOAWWhOAAVl4PxuF4VBOAUyxq4UJYVh9EEeFIAhNDbhYAGsQE78P/FXDZV38SR/DMABOdeuF4/ROEkXgWCXjYA40fwpIPsxVzMSQd40KSd53wJe60UgB44XgFBARIZ4cC0AsOAsAkBoBJHQOI5BKAQKSFA%2BIwApBmD4HQIkxA/4QGiLPUgMJmDEFeJwKeeCGivAAPLRG0NUQBU8IFsEEGQhgtBCFAN4FgaIXhgBuDELQP%2BfdSBYErEYcQrCBF4BYjUb0fDP6HmqOlNYn9lSYA7qI0M0RfgEI8FgHBPw8Bn34d6Yg0RUiYAFMSQwwBQxGFngsKgBhgAKAAGp4EwAAdzIUyIhvB%2BCCBEGIdgUgZCCEUCodQojdAuwMNY0ww9LD6DwNEP%2BkAFjZUcPuTgABaHEvoYlWEsGYDQewMlkLMP3QxxA8BYCSS1WwyjqEZBcIGUYfgXYhCmCUMoegUhpDSZkTwLQun5F6b0DpAwXbtF6V0EY/ScjjLqR0BgUzJjFH6PEcZExml6FtI0EZqyJALFHssVY%2Byj4cG7qQD%2B/dODSykv4DJm89jAGQK5KQAczB7GwvgD6PENhcDmLwQBwCFgIEwJbAYNSVEn1IGfLeAc95cCkque%2Bq5O4aB3lwDQLtLlf04L/f%2B08bELxAK/OFr8yXkrJauU5Gwe44O/gCwlpBDFpGcJIIAA%3D%3D"></iframe>

And we get identical assembly yet again (I've left it on `-O2` because the SIMD 
instructions are more succinct, but they also generate the same code on `-O1`).

Let's switch it up a bit, by comparing big-endian addition with little-endian 
addition.

<iframe width="800px" height="400px" src="https://godbolt.org/e?readOnly=true&hideEditorToolbars=true#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGEgKykrgAyeAyYAHI%2BAEaYxCAAbFykAA6oCoRODB7evgGp6ZkCoeFRLLHxSbaY9o4CQgRMxAQ5Pn5cgXaYDlkNTQQlkTFxickKjc2teR22EwNhQ%2BUjSQCUtqhexMjsHOYAzGHI3lgA1CZ7bsjj%2BILn2CYaAIL7h8eYZxcEAJ4pmAD6BGITEICjuD2eZgODCOXlO5zc0UIYKe4K8GSMJ2YbAUKSY2xO13OVhRTwImBYKQMZI%2BblceEMJ2wpBO31%2BWPeABVmazGKx3thmNF6B8ACIEgjoEAgXlC/54KgA%2BHXKV4BR/MJk4BA2h/ABu8I5YL292NJyOTAUChOdMMGqJ4PB4yYjmQfwtSmaEBtfhAzEcuve5xFQetDHwhiltEIBGFJFD4Z9frwAdFIe9UsRwFI4JOubz%2Bfz5jMAFk8KpMOh4/SGCcWHiEAsrU13gxUAQCV4UmlmhWiyt7STHmSKVTAxdvYzkY9zZaq7bbhcmScvBquAlFaaTAB2YnPJ75p0us0CcaYVQpYjL1fr9tEP7YCArwRrgEnXViFZnHc5gu5%2BXHhhT3PS8oGwVM9jFdMQEzE5DErKCkxTIM0zDatI2jehPzjH9f1w3CIDA5CILnH0owIGN3jgkipUQsdg2IqDMxWT9t13PD82ITACE2GsoHfWgzjMBITg0VQADExI0DQWONO4TgADi8FitzcHD2PwiB%2BME4TRKkiSZNUi4FKUgdHjw7dg33XDOO44ga340zCy3Sy9zMvNDzwZAAKAi8r2fG8TioYhUBYe9H2vV9%2BOUtjrK4niWVQMKoqJE4AHpUuPFgWC8Rp/XeVAUjAMA1Ish0nhSLwhU8kA1KfAgX3bfivEwRy829DUIE/EA3zEZqIGkr8rGctT2sEcL/Mij8Tm6prMAgO8Hyi6LSsHfNRoIcxhIKuJnRIINxvqgKosGtT81m0UEqSj9Wt/Gz4oAKgIBtQT2GLcxW1z822oEiEvOqGs67z21Y068zuuzAuC0LFt6zB%2B1ekrhsHUqEcHNEwmAE56DXC71vhKCyIo5l/pvI1d3RjEvBxi6SY3cn0Ux2IcZQhM7XHVCI2gvAsz8w6NxNXdHUBLwHBOFICD%2BbHhJBqzcylk5VGZL5TJRwWnnGYgRfbcW/ip6Xv1l5cccVk5ldR55hvN8F5aYdB0ElzA1wgG3mXl6JlsN8GayYaxohVpHXKZ4TbftoOICD2DmQj92Ts9uKIZ9yw/at5yODWWhOH8Xg/G4XhUE4VTLGsAkNi2QNIR4UgCE0NO1gAaxAfwzAAOgSLc9i3BJJASMwAE4O64IT9E4SReBYRu9mbjQEnkwezC3MxJF7jR5N73uElIHOtFIfOOF4BQQA0Kua7WOBYCQNAKToOJyEoS%2BUmv%2BJgCkMw%2BDoMliAP8Oa9IRFmGIL4nBK5/yaF8AA8tEbQ3Rq651IJfNgggwEMFoIAjg28sDRC8MANwYhaAH1gVgOsRhxBoN4PgTiPQAz4O3meboOUdjbw1DUH%2BUZohAgAR4LAP9AR4HHrAgMxBojpEwCKckhhgBRiMCfPgBhgAKAAGp4EwAAdzAWyIBvB%2BCCBEGIdgUgZCCEUCodQpDSC6GSAYKRpgi6WH0HgaIB9IBrAKnUQCnAAC0hIILWKsJYMwGgTjuLAWYPOAjiB4CwI4zq1RahZBcGGKY7QghhkGGUCoeg0gZFcYkjJhRXGpOGPEMYNRoG9DmDk4psT6hzAKUsIpsx%2BgVIac0Wp6SuBrAUKXbYEh06Z2zj/XeCt5IJHcV3E4wBkBeSkM3MwJwCL4F%2BoJPY7TeAwK0MxUgCBMC2xGNEjOHBR6kHHt3Zu/cuDyS3AvLc/gNC9y4BoZIW886cH3ofY%2BpCNkNzXqctevy/m/K3MPDgex%2BmmN3qsk%2BawBEZGcJIIAA%3D%3D%3D"></iframe>

Note the `rol` instructions, which rotate si and di by 8, effectively flipping 
the endianness. For 32/64 bit integers, this is replaced with `bswap`.

For the other side of the coin, let's look at a big-endian native system such
as MIPS. To keep things simpler, I've used the 32-bit version to remove extra 
shift right instructions the compiler inserts for widening 16-bit integers, 
keeping the assembly simpler to focus on the moving parts. I'm also adding a 
little-endian number to a big-endian number, and returning a big-endian number.

<iframe width="800px" height="600px" src="https://godbolt.org/e?readOnly=true&hideEditorToolbars=true#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIAKwapK4AMngMmAByPgBGmMQSABykAA6oCoRODB7evgFBaRmOAmER0SxxCVzJdpgOWUIETMQEOT5%2BgbaY9sUMjc0EpVGx8Um2TS1teZ0KE4PhwxWj1QCUtqhexMjsHOYAzOHI3lgA1CZ7bsiz%2BILn2CYaAIL7h8eYZxcEAJ4pmAD6BGITEICjuD2eZgODCOXlO5zcMUIYKe4K8GSMJ2YbAUKSY2xO13OVhRTwImBYKQMZI%2BblceEMJ2wpBO31%2BWPeABVmazGKx3thmDF6B8ACIEgjoEAgXlC/54KgA%2BHXKV4BR/cJk4BA2h/ABu8I5YL292NJyOTAUChOdMMGqJ4PBsyYjmQfwtShaEBtfhAzEcuve5xFQetDHwhiltEIBGFJFD4Z9frwAdFIe9UsRwFI4JOubz%2Bfz5jMAFk8KpMOh4/SGCcWHiEAsrc13gxUAQCV4UmkWhWiyt7STHmSKVTAxdvYzkY9zZaq7bbhcmScvBq9mZFaaTAB2YnPJ75p0us0CWaYVQpYjL1fr9tEP7YCArwRrgEnXViFZnHc5gu5%2BXHhhT3PS8oGwVM9jFdMQEzE5DErKCkxTIM0zDatI2jehPzjH9f1w3CIDA5CILnH0owIGN3jgkipUQsdg2IqDMxWT9t13PD82ITACE2GsoHfWgzjMAA2E4NFUAAxcSNGkmSvBY407hOMxJDkr83Bw9j8IgfjBJEsTpMkmSNFUo0wJORITK3dT900vC%2BLEXTRNUIzDOM%2BT1IuXMLJYqyNNs/N7IE8w9OcoyNEkkyLnhJSVP7PY2Nw7dgxs39OO44ga34gdHkLLdkr3HK80PPBkAAoCLyvZ8bxOKhiFQFh70fa9X34nyEtSrieJZVBGtaokTgAegG48WBYLwmn9d5UBSMAwA0pKHSeFIvCFEqQA0p8CBfdt%2BK8TBsvzb0NQgT8QDfMQ9ogDQ2oWlKSOOzbtvO2hTuey67wfVqbryxbCtzI7bmEk5pviZ0SCDJqqpaj8v3agtdro7reo/A7cLSrqACoCAbUF4vmn7B3zEGgSIS9HpvE6yvbVi/LzdGMpquqGs%2Bi7MDiuHboKha8cHNFwmAE56DXUV7oXWlUIjEAyIo5lyY3E1dz5jE4mFlCEztccJZ9TNZeaggjV3cEVbMWD0HQP4hbMCBjdg5lLZOGIbru%2BmayYawYmyzmjcwYWmDNv5jetn2TaYZkbcd2GNJd2D3c9gnHg4NZaE4fxeD8bheFQTh1MsawCQ2LZA0hHhSAITRE7WABrAIzAAOiErc9i3ITJCEswAE4m64YT9E4SReBYAI9lrjQhMSbuzC3ZT240RJ2/boTSHTrRSCzjheAUEAgjLjgtDWOBYCQNAKToeJyEoY%2BUlPhIWDwFIFGAKQzCCGhaDJYhN%2Bt8vSERZhiC%2BTgJdf7NC%2BAAeRiNoOoO8S7HzYIIUBDBaAAN3rwLAMQvDADcGIWgm8M6kCwHWIw4gUH4LwJxeoAZcErzPHUcaOwV4am6N/KMMQgT/w8Fgb%2BgI8CDzwQGYgMR0iYBFOSQwwAoxGHLmsKgBhgAKAAGp4EwAAd1AWyQBvB%2BCCBEGIdgUgZCCEUCodQJDdBcH0GIlA1hrD6DwDETekA1jTV6LggAtISCCphc6WGficVxoCzCZ34cQPAWAHEnS6D0LILgwxTD8OY0ICxyiVD0IUTIAg4mpPSOkhgQxknLEiVAhocxMnmNqPUAQ/QWh5JGFUcYAxSn1OqUk2pEg1gKALtsNpvcOCpyXt/NetY75WmAMgUqUha51w0CcAi%2BBSaCT2FwFYvAd57zWAgTAftRgROThwfupBB6t1rp3aoW4p5bkCO3LgGhzHL0zpwDeW9S5SKriAeexz56fK%2BZ8rcPS9hpwGQ855KDmKkH4RkZwkggA%3D%3D%3D"></iframe>

Unfortunately, MIPS doesn't have a native byteswap instruction being RISC, and 
you can see it breaking the shifts down instruction by instruction: `$6` stores 
`b`, and it is shifted by 24, -24 and -8 and put into `$7`, `$3` and `$2` 
respectively. We then `and` it together with the masks, noting that the `ff0000`
mask is too large to fit into the 32-bit instruction and has to be loaded into 
`$2` instead. Once the bitflip is done, we finally add them together.

This is a tricky example, so feel free to play around with it in godbolt. MIPS
calling conventions are not very well documented online, but from what I can 
make out, `$4` is the return register, `$5` and `$6` are arguments, and the 
`sw` after the `jr` return is some pipelining optimization, if I recall 
correctly from my uni architecture classes.

## Conclusion

With a working library that achieves my requirements in place, two questions 
still remain:

1. **Have I looked at other endian compatibility libraries, such as `boost::endian`?**:
   Yes, and they do very similar things. `boost::endian` splits it's 
   implementation across three main files:

   * [`conversion.hpp`][conversion]: includes [`endian_load.hpp`][endian_load]
   and [`endian_store.hpp`][endian_store]. These impelement the byteswaps for
   varying sizes of integers (more than I have supported)
   * [`buffer.hpp`][buffer]: represents unaligned bytes i.e. buffers for moving
   around ints in the library
   * [`arithmetic.hpp`][arithmetic]: overrides the operators to perform
   endian-specific arithmetic across types

   This is probably a better fit if you don't have access to a C++20 compiler,
   or don't want to roll out your own endian library. However it is too many
   moving parts for my taste, and plugging it into my project would take the
   challenge and fun away from crafting it by myself.

2. **What about supporting integer types?** This is a good question. However,
   then I would also need to manage underlying integer representations. Just as
   little endian machines are ubiquitous, two's complement machines are even
   more so. Yet, because we are accounting for the edge case which is
   big-endianness here, I would also need to handle more esoteric representations
   such as one's complement/sign bits in order to consider my implementation
   complete. It also doesn't make sense to implement them for my usecase, which 
   would mostly use this for unsigned counters/raw data in virtio.

The complete implementation, for including can be found [here][full-impl], and
is released under MIT. Feel free to point out errors, tests or other things I 
should analyse.


[1]: https://docs.oasis-open.org/virtio/virtio/v1.3/virtio-v1.3.html
[2]: https://waterjuiceweb.wordpress.com/2017/12/02/setting-up-a-virtual-big-endian-environment/

[conversion]: https://www.boost.org/doc/libs/1_85_0/boost/endian/conversion.hpp
[endian_load]: https://www.boost.org/doc/libs/1_85_0/boost/endian/detail/endian_load.hpp
[endian_store]: https://www.boost.org/doc/libs/1_85_0/boost/endian/detail/endian_store.hpp
[buffer]: https://www.boost.org/doc/libs/1_85_0/boost/endian/buffer.hpp
[arithmetic]: https://www.boost.org/doc/libs/1_85_0/boost/endian/arithmetic.hpp

[full-impl]: https://gist.github.com/neov5/f297723cbcdb5b80b2819cea739aa3af
