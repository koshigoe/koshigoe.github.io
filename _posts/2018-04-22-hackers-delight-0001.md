---
layout: post
title:  "Hacker's Delight #0001 Operating right end bits"
date:   2018-04-22 00:00:00 +09:00
categories:
- book
- Hacker's Delight
---


ref. Hacker's Delight ([English](https://www.amazon.co.jp/dp/B009GMUMTM) / [Japanese](https://www.amazon.co.jp/dp/443420159X))


Unset rightmost set bit
----

e.g. 01011000 -> 01010000

```
x & (x - 1)
```

The bits start from rightmost set bit are `10*`. (The `*` means repeat character more than 0.)
The reverse bits of `10*` is `01*`.
The `01*` is derived from `10* - 1`.

```
    0101 1000 (x)
AND 0101 0111 (x-1)
 -> 0101 0000
```

If `x & (x - 1)` is zero, the `x` is `2^n`.
(The `2^n` has just one set bit.)


2^n - 1
----

```
x & (x + 1)
```

If `x & (x + 1)` is zero, the `x` is `2^n - 1`.

```
0 = x         & (x + 1)
0 = (2^n - 1) & (2^n)
0 = 01{n}     & 10{n}

    01...
AND 10...
 -> 00...
```

(The `{n}` means repeat character exactly `n` times.)


Extract rightmost set bit
----

e.g. 01011000 -> 00001000

```
x & (-x)
```

The minus values is two's complement.

The operations to generate two's complement:

1. Reverse all bits
1. +1

The reverse operation generates `...01*`.
The +1 operation generates `...10*`.

```
    0101 1000
AND 1010 1000
 -> 0000 1000
```


Set only rightmost unset bit
----

e.g. 10100111 -> 00001000

```
^x & (x + 1)
```

The bits start from rightmost unset bit are `01*`.
And, `01*` plus `1` is `10*`.
The `^x` makes `{reversed bits}10*`.

```
    1010 1000
AND 0101 1000
 -> 0000 1000
```


The mask for trailing zeros
----

e.g. 01011000 -> 00000111


### A

```
^x & (x - 1)
```

Subtract `1` from `10*` is `01*`.
The `^x` makes `{reverse bits}01*`.

```
    10100111
AND 01010111
 -> 00000111
```

### B

```
^(x | -x)
```

The two's complement makes `{reverse bits}10*`.
The OR operation makes `1*0*`.

```
    01011000
 OR 10101000
 -> 11111000

NOT 11111000
 -> 00000111
```

### C

```
(x & -x) - 1
```

The `(x & -x)` is way for extract rightmost set bit.

```
    01011000 (x & -x)
 -> 00001000

    00001000 (-1)
 -> 00000111
```


The mask for bits start from rightmost set bit
----

e.g. 01011000 -> 00001111

```
x ^ (x - 1)
```

The `-1` operation reverse bits start from rightmost set bit.

```
    01011000
XOR 01010111
 -> 00001111
```


Set bits after rightmost set bit
----

e.g. 01011000 -> 01011111

```
x | (x - 1)
```

```
    01011000
 OR 01010111
 -> 01011111
```


Unset rightmost sequential 1
----

e.g. 01011000 -> 01000000

```
((x | (x - 1) + 1)) & x
```

The `x | (x - 1)` makes set bits start from rightmost set bit.
The `+1` reverse sequential 1 bits.

```
    01011000
    01010111
OR  01011111 x | (x - 1)

    01011111
 -> 01100000 + 1

    01100000
AND 01011000
 -> 01000000
```

If `((x | (x - 1) + 1)) & x` is zero, the `x` is `2^j - 2^k` (`j >= k >= 0`).
The both `2^j` and `2^k` have just one set bit.
So, `2^j - 2^k` makes `0*1*0*`.


Set rightmost unset bit
----

e.g. 10100111 -> 10101111

```
x | (x + 1)
```

The bits start from rightmost unset bit are `01*`.
And, `01*` plus `1` is `10*`.

```
    1010 0111
->  1010 1000 (+1)

    1010 0111
 OR 1010 1000
 -> 1010 1111
```


Iterate same size subsets
----

e.g. xxx011110000 -> xxx100000111 -> ...

```
s <- x & -x
r <- s + x
y <- r | (((x ^ r) >> 2) / s)
```

```c
unsigned snoob(unsigned x) {
    unsigned smallest, ripple, ones; // xxx0 1111 0000
    smallest = x & -x;               // 0000 0001 0000
    ripple = x + smallest;           // xxx1 0000 0000
    ones = x ^ ripple;               // 0001 1111 0000
    ones = (ones >> 2) / smallest;   // 0000 0000 0111
    return ripple | ones;            // xxx1 0000 0111
}
```
