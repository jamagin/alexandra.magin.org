---
type: blog
title: "Cryptopals 1-1: constant-time, for practice"
date: 2024-09-01
tags: ["cryptopals", "rust"]
draft: false
---

I happened to mention some cryptography-related things I was
learning in a Discord I'm in, and somebody again told me about
[Cryptopals](https://cryptopals.com/), and as a result I was nerdsniped into
looking at it, and in this case, starting to do it.

I'm not good at doing things the easy way when I know that doing
something in a more difficult way will help me learn something I want
to learn, so I've started out in Rust and I'm trying to write (at least
at the source level) constant-time code[^1]. I'm also trying to avoid
third-party crates in favor of my own code for low level manipulation
of data.

[^1]: Different sources may define this differently, but generally the idea
is that memory access and control flow shouldn't depend on the content
of data. [A beginner's guide to constant-time cryptography](https://www.chosenplaintext.ca/articles/beginners-guide-constant-time-cryptography.html) or [Why Constant-Time Crypto?](https://www.bearssl.org/constanttime.html) can give you some starting points. It's been known for 
at least two decades now that non-constant-time code enables side-channel
attacks, see [Brumley and Boneh, "Remote Timing Attacks are Practical" (pdf)](https://crypto.stanford.edu/~dabo/papers/ssl-timing.pdf).

Putting aside for a moment matters of idiomatic rust and usage of
the standard library (and error handling, illegal digits parse as 0),
how do we convert a character representing
a hex digit (in this case an ASCII character in a u8) to an unsigned
integer (also a u8), in keeping with the
[problem's suggestion](https://cryptopals.com/sets/1/challenges/1) that
we "Always operate on raw bytes, never on encoded strings.
Only use hex and base64 for pretty-printing"?

Here's an obvious approach:

```Rust
fn hex_u8_to_u8(x: u8) -> u8 {
    if x >= b'0' && x <= b'9' {
        return x - b'0'
    } else if x >= b'A' && x <= b'F' {
        return x - b'A' + 10
    } else if x >= b'a' && x <= b'f' {
        return x - b'a' + 10
    }
    0
}
```

If you've spent any time thinking about logical operators and if/else clauses,
you know what's wrong, but lets [put it in godbolt.org](https://godbolt.org/z/d8j3bvKaM)
(`rustc -C opt-level=1` but `#[inline(never)]` on the function):

```NASM
example::hex_u8_to_u8::h8e8fcdde4917722f:
        lea     eax, [rdi - 48]
        cmp     al, 10
        jb      .LBB14_4
        lea     eax, [rdi - 65]
        cmp     al, 6
        jae     .LBB14_2
        add     dil, -55
        mov     eax, edi
.LBB14_4:
        ret
.LBB14_2:
        lea     eax, [rdi - 97]
        add     dil, -87
        xor     ecx, ecx
        cmp     al, 6
        movzx   eax, dil
        cmovae  eax, ecx
        ret
```

Eww. There's two branches, and we return in two different places. That doesn't work.
So, here's my reasonably-readable but constant-time (at least on this compiler
version and architecture version):

```Rust
fn hex_u8_to_u8(x: u8) -> u8 {
    let is_letter = (((x >= b'A') & (x <= b'F')) | ((x >= b'a') & (x <=b'f'))) as u8;
    let letter_off = (x & 0b0000111) + 9;
    let is_digit = ((x >= b'0') & (x <= b'9')) as u8;
    let digit_off = x & 0b0001111;
    is_letter * letter_off + is_digit * digit_off
}
```

Again [in godbolt.org](https://godbolt.org/z/d4n6rWP77):

```NASM
example::hex_u8_to_u8::h8e8fcdde4917722f:
        mov     eax, edi
        and     al, -33
        add     al, -65
        mov     ecx, edi
        and     cl, 7
        add     cl, 9
        lea     edx, [rdi - 48]
        and     dil, 15
        xor     esi, esi
        cmp     al, 6
        movzx   ecx, cl
        cmovae  ecx, esi
        cmp     dl, 10
        movzx   eax, dil
        cmovae  eax, esi
        add     al, cl
        ret
```

That looks pretty nicely optimized! So, we have no branches and a single return.
Is this constant-time code? Probably. `add`/`and`/`xor`/`cmp` should all be.
`mov` between registers is. `lea` looks scary but it's just provided for array
indexing and used to do a simple ALU operation here, not a load from memory. So
what's `cmovae`? The optimizer has cleverly turned a multipication into a
conditional move. Is that constant-time? [Intel says we're
okay](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/secure-coding/mitigate-timing-side-channel-crypto-implementation.html):

> The CMOVcc instruction runs in time independent of its arguments in all current x86 architecture processors. This includes variants that load from memory. The load is performed before the condition is tested.

Well, that was a lot to merely verify that the fragment of code
that converts characters of the input
([full source](https://github.com/jamagin/cryptopals/blob/806d2cfad5847f4fda1ddaa8c8c788d14ba25723/src/util.rs#L4))
is constant time (with a specific compiler and architecture). In the
future, I'd like to look at the rest of the code, and I'd really
like to do idiomatic Rust error handling of invalid input without
destroying the constant-time property of this code.

## Update, idiomatic Rust error handling

After looking at some other Rust I was pointed to, such as [base16ct](https://docs.rs/base16ct/latest/src/base16ct/lib.rs.html#116)
I tried to add some error handling to that function ([full source](https://github.com/jamagin/cryptopals/blob/874714562d36ba8d61e9a0a6538ccf4a87daa8c0/src/bytes.rs#L6)) and it seems like
this did fine:

```Rust
#[derive(Debug, PartialEq)]
pub struct HexParseError;

fn hex_u8_to_u8(x: u8) -> Result<u8, HexParseError> {
    let is_letter = (((x >= b'A') & (x <= b'F')) | ((x >= b'a') & (x <=b'f'))) as u8;
    let letter_off = (x & 0b0000111) + 9;
    let is_digit = ((x >= b'0') & (x <= b'9')) as u8;
    let digit_off = x & 0b0001111;
    let output = is_letter * letter_off + is_digit * digit_off;
    match is_letter | is_digit {
        0 => Err(HexParseError),
        _ => Ok(output),
    }
}
```

```NASM
example::hex_u8_to_u8::h016b68b3d3772cbd:
        mov     eax, edi
        and     al, -33
        add     al, -71
        lea     ecx, [rdi - 58]
        mov     esi, edi
        and     sil, 7
        add     sil, 9
        and     dil, 15
        xor     r8d, r8d
        cmp     cl, -10
        setb    cl
        movzx   edx, dil
        cmovb   edx, r8d
        cmp     al, -6
        setb    al
        movzx   esi, sil
        cmovb   esi, r8d
        and     al, cl
        add     dl, sil
        ret
```

I decided to run `cargo clippy` and it suggested that I change those
range comparisons to `(a..b).contains(&x)` and `.is_ascii_digit()` which looked
like it might ruin our day:

```Rust
fn hex_u8_to_u8(x: u8) -> Result<u8, HexParseError> {
    let is_letter = ((b'A'..=b'F').contains(&x) | (b'a'..=b'f').contains(&x)) as u8;
    let letter_off = (x & 0b0000111) + 9;
    let is_digit = x.is_ascii_digit() as u8;
    let digit_off = x & 0b0001111;
    let output = is_letter * letter_off + is_digit * digit_off;
    match is_letter | is_digit {
        0 => Err(HexParseError),
        _ => Ok(output),
    }
}
```

I won't repeat the assembly, because it gave me the same output.
I'm impressed.
