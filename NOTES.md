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

- Which SLH-DSA variant does libbitcoinpqc actually build, SHA2-128s
  or SHAKE-128s? The docs say both in different places. Matters for
  test vectors.
- FIPS 206: revisit when the final standard lands.
- MAX_SCRIPT_ELEMENT_SIZE (520 B) vs multi-kB signatures: witness
  element chunking, or a new limit under a new leaf version? Needs a
  decision note before any code.
- Sighash: reuse the BIP 341/342 message unchanged (P2MR already
  does), or extend it for PQ? Check what libbitcoinpqc's QuBit code
  assumes.
- Weight: my validation weight budget test (bitcoin-p2mr, FUTURE item
  8) is the measurement harness; port it to the new branch.

## Next steps

1. Clone and build libbitcoinpqc, read its API and QuBit integration
   assumptions, settle the SHA2/SHAKE question.
2. Create the `pq-opcode` branch off `p2mr-regtest`.
3. Write the opcode design note (leaf version, scheme byte, encoding,
   520 B handling) before any consensus code.
4. Prototype the verify path for one scheme end to end, then the
   second.
5. Measure witness sizes and validation cost; comparison table
   including paper numbers for FN-DSA, SHRINCS and SQIsign.
