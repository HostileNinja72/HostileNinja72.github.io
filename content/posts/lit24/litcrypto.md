---
weight: 1
title: "LIT CTF 2024 - Crypto writeups"
date: 2024-08-11T16:37:00+06:00
lastmod: 2024-08-11T16:37:00+06:00
draft: false
author: "HostileNinja72"
authorLink: "https://HostileNinja72.github.io"
description: "LIT Crypto Writeups"

tags: ["crypto", "LIT CTF", "RSA", "coppermsith", "modular arithmetic"]
categories: ["Writeups"]

lightgallery: true

math:
  enable: true

toc:
  enable: true
---

Writeups for some Cryptography challenges in LITCTF 2024.

<!--more-->

## Symmetric RSA

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
#### Code Analysis
We automatically notice the abnormal use of $e$ as the prime $p$.

The code gives us the encryption of the flag as the first `CT`, then it gives us the possibility to encrypt 4 inputs of our choice.

#### Recovering p

We send 2 intputs, $2$ and $3$ and we get: 

$$
\begin{aligned}
ct_{2} = 2^p \pmod{n} \\\
ct_{3} = 3^p \pmod{n}
\end{aligned}
$$

By applying **Fermat's little theorem**:
$$
\begin{aligned}
2^p \equiv 2 \pmod{p} \\\
3^p \equiv 3 \pmod{p}
\end{aligned}
$$

after substracting we get:
$$
\begin{aligned}
ct_{2} - 2 \equiv 0 \pmod{p}
ct_{3} - 3 \equiv 0 \pmod{p}
\end{aligned}
$$


Since both values are divisible by $p$, then $p$ must be :

$$
gcd(ct_{2} - 2, ct_{3} - 3) = p
$$

#### Finding n

For n we can find it with:
$$
\begin{aligned}
-1 \equiv n - 1 \pmod{n} 
\end{aligned}
$$
Our $p$ is `odd`, as we send -1 to the server we will get:

$$
\begin{aligned}
ct_{-1} = (-1)^p \pmod{n} \\
\therefore ct_{-1} \equiv -1 \equiv n - 1 \pmod{n}
\end{aligned}
$$

Meaning we can extract n by adding `1` to the output.

#### Solution Code
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

> FLAG : **LITCTF{ju57_u53_e=65537_00a144ca}**

## Truly symmetric RSA

This challenge has the same concept as before, $e = p$.

Here is The script:
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
We are giving $ct$ as the encryption on the flag, with the length of the flag `len(PT)`, and the value of $n$.

Giving this setup, we will think of using the coppersmith attack. The encryption is done following this equation:

$$ \begin{aligned} ct =  flag^p \pmod{n} \\ \end{aligned} $$

hence:
$$ \begin{aligned} ct =  flag^p \pmod{p} \\ \end{aligned} $$

By applying **Fermat's little theorem** we have:
$$ \begin{aligned} flag^p \equiv flag \mod{p} \end{aligned} $$
so we can write:
$$
\begin{aligned}
ct \equiv flag \mod{p}
\end{aligned}
$$

Now we apply the coppersmith's method. We define a polynomial $f(\text{flag}) = \text{flag} - \text{CT} $ over the ring  $ \mathbb{Z}/n\mathbb{Z}$. We seek a small root 
$$
f(\text{flag}_0) \equiv 0 \mod n \quad \text{with} \quad \text{flag}_0 < X
$$

#### Solution Code
We apply this in a sage code:
```python 
from sage.all import *
from Crypto.Util.number import long_to_bytes

CT = 155493050716775929746785618157278421579720146882532893558466000717535926046092909584621507923553076649095497514130410050189555400358836998046081044415327506184740691954567311107014762610207180244423796639730694535767800541494145360577247063247119137256320461545818441676395182342388510060086729252654537845527572702464327741896730162340787947095811174459024431128743731633252208758986678350296534304083983866503070491947276444303695911718996791195956784045648557648959632902090924578632023471001254664039074367122198667591056089131284405036814647516681592384332538556252346304161289579455924108267311841638064619876494634608529368113300787897715026001565834469335741541960401988282636487460784948272367823992564019029521793367540589624327395326260393508859657691047658164
n = 237028545680596368677333357016590396778603231329606312133319254098208733503417614163018471600330539852278535558781335757092454348478277895444998391420951836414083931929543660193620339231857954511774305801482082186060819705746991373929339870834618962559270938577414515824433025347138433034154976346514196324140384652533471142168980983566738172498838845701175448130178229109792689495258819665948424614638218965001369917045965392087331282821560168428430483072251150471592683310976699404275393436993044069660277993965385069016086918288886820961158988512818677400870731542293709336997391721506341477144186272759517750420810063402971894683733280622802221309851227693291273838240078935620506525062275632158136289150493496782922917552121218970809807935684534511493363951811373931

PR.<flag> = PolynomialRing(Zmod(n))

f = flag - CT

X = 2**(8*62) # len(flag) is 62
beta = 0.41
roots = f.small_roots(X=X, beta=beta)


recovered_flag = long_to_bytes(int(roots[0])).decode()
print("Recovered flag:", recovered_flag)

```
>FLAG : **LITCTF{I_thought_the_bigger_the_prime_the_better_:(_72afea90}**
















