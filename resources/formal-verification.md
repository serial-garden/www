# Formal Verification

## Cryptol

\*\*\*\*[**Web**](https://cryptol.net/) \| [**GitHub**](https://github.com/galoisinc/cryptol)\*\*\*\*

Domain-specific language and runtime for specifying cryptographic algorithms. Can be used in conjunction with other tools like [SAW](formal-verification.md#software-analysis-workbench-saw) to formally verify specific source or binary implementations.

```text
// https://cryptol.net/#cryptol-sha-1-implementation
f : ([8], [32], [32], [32]) -> [32]
f (t, x, y, z) =
  if (0 <= t)  && (t <= 19) then (x && y) ^ (~x && z)
   | (20 <= t) && (t <= 39) then x ^ y ^ z
   | (40 <= t) && (t <= 59) then (x && y) ^ (x && z) ^ (y && z)
   | (60 <= t) && (t <= 79) then x ^ y ^ z
   else error "f: t out of range"
```

## Software Analysis Workbench \(SAW\)

\*\*\*\*[**Web**](https://saw.galois.com/) \| [**GitHub**](https://github.com/GaloisInc/saw-script)\*\*\*\*

Scripting language \(SAWScript\), interpreter, and collection of tools for formally verifying C, Java, and [Cryptol](formal-verification.md#cryptol) code.

```text
// https://github.com/GaloisInc/saw-script/blob/master/examples/cryptol-2/des-cryptol2.saw
m <- cryptol_load "../../deps/cryptol-specs/Primitive/Symmetric/Cipher/Block/DES.cry";
let enc = {{ m::DES.encrypt }};
let dec = {{ m::DES.decrypt }};
let {{ thm key msg = dec key (enc key msg) == msg }};
prove_print abc {{ thm }};
```



