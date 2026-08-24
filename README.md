# blake3

BLAKE3 for [sysl](https://github.com/sysl-lang/sysl) — hashing, keyed hashing and key derivation,
bound to the [official C implementation](https://github.com/BLAKE3-team/BLAKE3).

```sysl
import sh.sysl.blake3.{blake3, keyed, derive_key}
import sysl.encoding.hex_string

val digest = blake3("abc".bytes)

print(hex_string(digest))
```

**Nothing has to be installed to use this.** This package carries the three C files it needs and sysl
compiles them as part of the build, so there is no `-l` flag, no `pkg-config`, no CMake and no build
script anywhere in this repository.

## One function, three modes

BLAKE3 is a single compression function reached three ways, and which one you are in is fixed when the
hasher starts rather than when it finishes.

| | |
|---|---|
| `blake3(data)` | an ordinary hash, answering 32 bytes |
| `keyed(key, data)` | a message authentication code, keyed with exactly 32 bytes |
| `derive_key(context, material, out)` | a key of any length, derived for one hardcoded purpose |

**The three are domain-separated**, which is why they are separate entry points rather than
conventions a caller applies: the same message under the same key gives unrelated output in each mode,
so a value derived for one purpose cannot be replayed as another. There is a test that says so.

`context` must be hardcoded and globally unique to one use — upstream's advice is an application name
with a date in it, `"myapp 2026-08-24 session keys"`. It is not a password, not a salt, and not
anything supplied at run time.

## The output has no natural length, and it is seekable

Thirty-two bytes is the default and what the one-shots answer, but `fill` writes as many as you give
it room for and `fill_from` starts anywhere in that stream:

```sysl
var keystream: []u8 = [0; 4096]

var h = keyed_hasher(key)

h.push(nonce)
h.fill_from(offset, keystream)
```

The ten-thousandth byte costs no more than the first, which is what lets BLAKE3 stand in for a
keystream, or verify one block of a large file without producing the blocks before it.

## Nothing here asks you for storage

A hasher is 1,912 bytes, and every other caller-placed binding in this organisation would have made
that your problem — miniz's `Deflater` is *placed* in a slice you own, because sysl has no allocator
to reach for.

**This one does not have to, because a `blake3_hasher` holds no pointers.** It is a key, a chunk state
and a stack of chaining values, all inline, so it can be copied from one address to another and keep
working. That lets the state live inside a `Hasher` as a field, which makes a `Hasher` an ordinary
sysl value: it goes in a `var`, it can be passed to a function and returned from one, and there is no
storage argument and no alignment to get right.

The `a hasher survives being moved` test is what pins that reading, and it is why it exists.

The state is declared as an array of `u64` rather than of bytes, and that is alignment's whole answer:
the chunk state holds a `uint64_t` counter, so the struct wants an 8-aligned address, and an array of
`u64` has one by construction where a `[1912]u8` would promise nothing. The raw layer asserts
`_Alignof(blake3_hasher) == 8` so that a change on the C side fails a test rather than corrupting a
digest.

## It is compiled portable-only, and that is a real cost

BLAKE3 is fast because it vectorises — upstream ships SSE2, SSE4.1, AVX2, AVX-512 and NEON
implementations alongside the portable one, and expects a build script to detect the CPU and hand the
right files to the compiler.

**sysl compiles a package's C declaratively and runs no build script**, so the choice is made in
`package.hocon` instead: portable everywhere, with only the three portable `.c` files vendored.

```hocon
defines {
  "sh/sysl/blake3/c/{blake3,blake3_dispatch,blake3_portable}.c" {
    BLAKE3_NO_SSE2   = true
    BLAKE3_NO_SSE41  = true
    BLAKE3_NO_AVX2   = true
    BLAKE3_NO_AVX512 = true
    BLAKE3_USE_NEON  = 0
  }
}
```

**`BLAKE3_USE_NEON` is the one that is easy to miss.** It is not an x86 macro and none of the four
covers it: `blake3_impl.h` autodetects it to **1** on any aarch64 target — a Mac, a phone, a Pi — so a
build carrying only the four x86 macros links against a `blake3_hash_many_neon` that is not here.

It is set to 0 rather than answered by vendoring `blake3_neon.c`, because that file includes
`<arm_neon.h>` unconditionally and guards none of its body: carrying it would break every target
without NEON, which includes the RP2350 this organisation builds for.

So on a large input this is slower than a `b3sum` built by upstream's own build system. It is still
BLAKE3, and still a tree hash — the vectors below are the published ones and they pass. If a
host-only consumer ever wants the SIMD paths, the honest answer is a second package rather than a
conditional in this one.

## What it asks of the machine

No allocator, no clock, no file — there is no `malloc`, `calloc` or `free` in any of the three C files,
which is what makes the caller-owned arrangement possible at all. So `package.hocon` declares no
capabilities.

**That is a claim about what this package asks of sysl, and not a claim that it builds for a bare
target.** `blake3_impl.h` includes `<assert.h>` and `<string.h>`, and clang gives a freestanding target
neither:

```
sysl build-lib . --target thumb-freestanding
fatal error: 'assert.h' file not found
```

Any target with a libc is fine, and that covers the interesting ones: the Pico SDK's newlib, Zephyr,
and a FreeRTOS application. The sibling [`sha3`](https://github.com/sysl-lang/sha3) package is the one
that genuinely needs nothing.

## The tests are somebody else's numbers

Every expected value in `tests.sysl` is from the BLAKE3 team's published `test_vectors.json`, which is
generated from the **Rust** reference implementation — while the code under test is the **C** one. So
these are cross-implementation checks rather than a binding compared against itself.

**It is one oracle rather than two, and that is worth naming.** Neither `b3sum` nor a Python BLAKE3 is
present on this machine, and installing one would write into a Homebrew tree this account does not
own. The published vectors are what the algorithm's authors publish, which is the standing an RFC's
vectors have.

The lengths are chosen for where a tree hash actually breaks: 63/64/65 straddle a block, and
1023/1024/1025 straddle a **chunk** — everything below 1,024 bytes hashes as a lone chunk and
exercises none of the tree, so a chaining bug is invisible until the second one. 2,048 through 8,192
cover the parent nodes above that.

```
21 passed, 0 failed
```

## Layout

```
sh/sysl/blake3/
    blake3.sysl         the binding
    tests.sysl          the published vectors and the behavioural tests
    c/
        c.sysl          BLAKE3 as C declares it
        blake3.c        vendored from BLAKE3-team/BLAKE3 at f3149ec, unmodified
        blake3_dispatch.c
        blake3_portable.c
        blake3.h
        blake3_impl.h
        tests.sysl      the ABI and specification facts the layer above depends on
package.hocon
```

**Everything that is C lives in `c/`**, module `sh.sysl.blake3.c`. That is the two-layer shape every
binding in this organisation uses: the `c` layer has to be *faithful*, and for a cryptographic library
a signature that disagrees with the header links perfectly and then reads past the end of a key. The
layer above it has to be *pleasant*, which is a different question.

**There is no `shim.c`.** Every entry point is an ordinary exported function — BLAKE3's header declares
no macros and no `static inline` — so there is nothing sysl cannot reach directly. The vendored C is
unmodified.

**A key's length is enforced in the pleasant layer, because C cannot enforce it.**
`const uint8_t key[32]` is a pointer the compiler checks nothing about, so `keyed_hasher` takes a slice
and `require`s its length — which is the only place the 32 is checked rather than described.

## Using it

Name it in your project's `package.hocon` and `sysl build` fetches it:

```hocon
dependencies {
  blake3 { git = "github.com/sysl-lang/blake3", version = "0.1.0" }
}
```

## Licence

BLAKE3's C implementation is dual-licensed CC0-1.0 and Apache-2.0 (with an LLVM-exception variant);
`LICENSE_CC0`, `LICENSE_A2` and `LICENSE_A2LLVM` are upstream's, carried unchanged. The binding is
offered under the same terms.
