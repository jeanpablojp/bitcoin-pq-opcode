# PQ opcode: working notes

The goal is to prototype the "TBD Post Quantum Signature BIP" that
BIP 361 requires and BIP 360 leaves open (line 338: the authors are
"currently researching options"). Code goes on branch `pq-opcode`,
stacked on `p2mr-regtest` in my Core fork. This repo is the journal,
same format as bitcoin-p2mr.

## Recon 2026-08-02 (initial pass)

### Scheme landscape

| scheme | sig | pubkey | status | verdict |
|---|---|---|---|---|
| ML-DSA-44 | 2420 B | 1312 B | FIPS 204 final | candidate (level 2) |
| SLH-DSA-128s | 7856 B | 32 B | FIPS 205 final | candidate (level 1) |
| FN-DSA/Falcon | ~666 B | ~897 B | FIPS 206 draft, final expected late 2026/early 2027 | table only, revisit at final |
| SHRINCS/SHRIMPS | ~2.5 kB | - | research (eprint 2025/2203, Kudinov/Nick); SHRIMPS is semi-stateful, posted on Delving | table only |
| SQIsign | ~0.2-0.3 kB | ~64 B | NIST additional round, years out | table only |
| XMSS/LMS | ~2.5 kB | small | SP 800-208, stateful | rejected (sipa: "stateful signature schemes are scary") |

### libbitcoinpqc

libbitcoinpqc (github.com/cryptoquick/libbitcoinpqc) is by cryptoquick
(Hunter Beast, lead author of BIP 360). It is the declared "crypto
foundation for the Bitcoin QuBit soft fork" and implements exactly
ML-DSA-44 + SLH-DSA-128s (vendored NIST reference code, MIT license,
~97 commits, not production-hardened, no audit). The chaincode report
already describes the future companion BIP as "expected to support
ML-DSA and SLH-DSA", so the scheme choice looks settled on those two.
No draft BIP text or opcode spec exists anywhere public as of
2026-08-02 (searched bitcoin-dev, Optech, Delving).

### Library options for the prototype

1. libbitcoinpqc: MIT, C, exactly the two candidate schemes, and the
   API is Bitcoin-oriented. Downside: young, unaudited.
2. liboqs (open-quantum-safe): mature and broad, also wraps reference
   code, but a heavier dependency than the problem needs.
3. Vendoring NIST reference code directly: what libbitcoinpqc itself
   does, and the most Core-like approach (Core vendors its crypto and
   avoids dependencies).

My current pick is libbitcoinpqc: it targets exactly the two candidate
schemes with a Bitcoin-oriented API, and exercising it end to end will
shake out issues worth reporting upstream. I want the opcode itself to
stay scheme-agnostic (a scheme byte in the design), so the backend can
be swapped without touching consensus logic if a better library or a
new scheme shows up.

### Open questions for next pass

- FIPS 206: revisit when the final standard lands.
- MAX_SCRIPT_ELEMENT_SIZE (520 B) vs multi-kB signatures: witness
  element chunking, or a new limit under a new leaf version? Needs a
  decision note before any code.
- Sighash: reuse the BIP 341/342 message unchanged (P2MR already
  does), or extend it for PQ? Check what libbitcoinpqc's QuBit code
  assumes.
- Weight: my validation weight budget test (bitcoin-p2mr, FUTURE item
  8) is the measurement harness; port it to the new branch.

## Hands-on with libbitcoinpqc, 2026-08-02

Cloned and built it on my machine, pinned at commit 053e954. All 3 of
its tests pass on macOS. What I learned:

- The SLH-DSA variant question is settled: the build uses
  SLH-DSA-SHA2-128s (PARAMS=sphincs-sha2-128s in CMake). Their PR #28
  switched from SHAKE-128s and the README documents the migration;
  the SHAKE naming that still shows up in the crates.io description
  is stale.
- Sizes confirmed with a real round trip against the built library
  (keygen, sign, verify): ML-DSA-44 pk 1312 B / sig 2420 B,
  SLH-DSA-SHA2-128s pk 32 B / sig 7856 B. secp256k1 is algorithm 0
  in the same API, handy for like-for-like comparison. One full
  SLH-DSA keygen+sign+verify cycle costs about 1.2 s of CPU on my
  machine.
- The API is a clean fit for a scheme-agnostic opcode: verify() takes
  (algorithm enum, pubkey, message, signature) over arbitrary bytes.
  The enum already numbers the algorithms (1 = ML-DSA-44, 2 =
  SLH-DSA-SHA2-128s), which maps naturally onto a scheme byte.
- No sighash or transaction logic anywhere in the library: it is pure
  crypto primitives. Whatever sighash the opcode uses is entirely a
  design decision on my side.
- The SLH-DSA test alone takes ~10 of the suite's 11.5 seconds.
  Hash-based signing/verification cost is real and will show up in
  the stage 5 measurements.

Scheme order for the prototype: SLH-DSA-SHA2-128s first, then
ML-DSA-44. The 7856 B signature stresses the 520 B witness element
question from day one, and the 32 B pubkey fits a leaf script the
same way a Schnorr key does today.

## Next steps

1. Write the opcode design note (leaf version, scheme byte, encoding,
   520 B handling) before any consensus code.
2. Prototype the verify path for SLH-DSA end to end, then ML-DSA.
3. Measure witness sizes and validation cost; comparison table
   including paper numbers for FN-DSA, SHRINCS and SQIsign.
