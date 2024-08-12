---
weight: 1
title: "LIT CTF 2024 - Symmetric RSA"
date: 2024-08-11T16:37:00+06:00
lastmod: 2024-08-11T16:37:00+06:00
draft: true
author: "HostileNinja72"
authorLink: "https://HostileNinja72.github.io"
description: "LIT Crypto Writeups"

tags: ["crypto", "LIT CTF", "RSA", "Asymmetric encryption"]
categories: ["Writeups"]

lightgallery: true

math:
  enable: true

toc:
  enable: true
---

Writeups for some Cryptography challenges in LITCTF 2024.

<!--more-->

# Symmetrc RSA

In the description of the challenge we are given a server to connect to with nc with a chall code:


```python
#!/usr/bin/env python3
from Crypto.Util.number import long_to_bytes as ltb, bytes_to_long as btl, getPrime

p = getPrime(1024)
q = getPrime(1024)

n = p*q

e = p

with open("flag.txt", "rb") as f:
	PT = btl(f.read())

CT = pow(PT, e, n)
print(f"{CT = }")

for _ in range(4):
	CT = pow(int(input("Plaintext: ")), e, n)
	print(f"{CT = }")

```
## Code Analysis
We automatically notice the abnormal use of $e$ as the prime $p$.

The code gives us the encryption of the flag as the first CT, then it gives us the possibility to encrypt 4 inputs of our choice.

## Recovering p

We send 2 intputs: $2$ and $3$
we get: 

$$
\begin{aligned}
ct_{2} = 2^p \pmod{n} \\\
ct_{3} = 3^p \pmod{n}
\end{aligned}
$$

We can notice that p gonna be:

$$
\begin{aligned}
gcd(ct_{2} - 2, ct_{3} - 3) = p
\end{aligned}
$$

We can proof this formula here with Fermat's little theorem and the Chinese reminder theorem.
### Proof

## Finding n

For n we can find it with:
$$
\begin{aligned}
-1 \equiv n - 1 \pmod{n} 
\end{aligned}
$$
as we send -1 to the server we will get:

$$
\begin{aligned}
ct_{-1} = (-1)^p \pmod{n} \therefore n - 1 = (-1)^p \pmod{n} 
\end{aligned}
$$

Meaning we can extract n by adding `1` to the output.

## Solution Code
We apply our logic in a code to get the flag:

```python
from pwn import remote
from math import gcd
from Crypto.Util.number import long_to_bytes as ltb

r = remote('litctf.org', 31783)

r.recvuntil("CT = ")
ct_flag = int(r.recvline().strip())
print(f"Initial CT (flag): {ct_flag}")

ciphers = []

for plaintext in [2, 3, -1]:
    r.sendline(str(plaintext))
    r.recvuntil("CT = ")
    ct = int(r.recvline().strip())
    ciphers.append(ct)
    print(f"CT for plaintext {plaintext}: {ct}")
r.close()

p = gcd(ciphers[0]-2, ciphers[1]-3) 
n = ciphers[2] + 1
q = n // p

phi = (p-1)*(q-1)
d = pow(p, -1, phi)

flag = ltb(pow(ct_flag, d, n))
print(flag)


```

and we get the following flag: `LITCTF{ju57_u53_e=65537_00a144ca}`

# Truly symmetric RSA

This challenge has the same concept as before, $e = p$.
The script:
```python
#!/usr/bin/env python3
from Crypto.Util.number import long_to_bytes as ltb, bytes_to_long as btl, getPrime

p = getPrime(1536)
q = getPrime(1024)

n = p*q

e = p

with open("flag.txt", "rb") as f:
	PT = f.read()

CT = pow(btl(PT), e, n)
print(f"{len(PT) = }")
print(f"{CT = }")
print(f"{n = }")

```
We are giving $ct$ as the encryption on the flag. With the length of the flag `len(PT)`, and the value of $n$.

The encryption is done following this equation:
$$
\begin{aligned}
ct =  pt^p \pmod{n} \\
\end{aligned}
$$








