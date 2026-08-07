# OP_CHECKPQSIG: design note

Design for the prototype on the `pq-opcode` branch. Every decision
lists the alternative I rejected and why. Nothing here is final spec;
it is the shape I am building and measuring. Both schemes are
implemented and tested.

The two schemes named below are the standardised ones. SLH-DSA comes
from slhdsa-c and ML-DSA from the Dilithium reference code, and the
SLH-DSA half is checked against NIST's own ACVP vectors inside the
test suite, which is what lets the name be used without qualifying
it. What the opcode signs is the FIPS 205 pure variant with an empty
context, so the message reaching the scheme is the domain separator,
a zero context length and then the 32-byte tapscript sighash.

## 1. Delivery: redefine an OP_SUCCESSx inside tapscript

The opcode is a redefinition of OP_SUCCESS187 (0xbb) under leaf
version 0xc0, the same leaves P2MR already executes with BIP 342
rules. The prototype gates it behind a script flag
(SCRIPT_VERIFY_PQSIG) that block validation applies on regtest only,
the same pattern SCRIPT_VERIFY_P2MR uses. Mempool policy applies it
everywhere, which the open questions below cover.

This works because of an ordering fact I verified in Core's
interpreter (ExecuteWitnessScript, interpreter.cpp): the OP_SUCCESSx
scan runs before every other check, including the 520-byte limit on
initial witness stack elements. Core's own comment reads "OP_SUCCESSx
processing overrides everything, including stack element size
limits". A soft fork that gives OP_SUCCESS187 semantics is therefore
free to define its own witness element size rules for scripts that
contain it.

This is also what makes the change deployable as a soft fork at all:
with the flag off, 0xbb stays OP_SUCCESS and the script is
anyone-can-spend, so every rule the new semantics add is a
restriction on something previously valid. That is the upgrade path
BIP 342 reserved the OP_SUCCESSx range for.

The flip side, and a reason a real BIP might prefer a different
OP_SUCCESS number or the pqdata route: reusing 0xbb quietly changes
what that byte means, so anything that treats the whole OP_SUCCESSx
range uniformly has to special-case it. Core's own feature_taproot.py
does exactly that, generating an OP_SUCCESS test per opcode value and
asserting the unconditional-success behavior; with the flag active in
regtest block validation, 0xbb no longer matches and the case has to
be skipped. That is a signal worth carrying into the spec discussion,
not just a test edit.

Rejected: a new leaf version. It would buy the same freedom at the
cost of specifying a whole execution context from scratch, and it
would throw away the BIP 342 machinery P2MR already reuses (sighash,
validation weight budget, script limits). Also rejected: the new
witness style / pqdata area being discussed on Delving (thread 2702).
That is the likely long-term home for multi-kB signatures, but it
needs a transaction serialization and P2P change that is still at the
design stage; a prototype should not block on it, and raw byte
measurements stay valid whatever costing that effort lands on.

## 2. Large elements: one element per object, no chunking

Signature and public key each travel as a single witness stack
element. Scripts whose executed script contains OP_CHECKPQSIG accept
initial stack elements up to PQ_MAX_ELEMENT_SIZE = 8192 bytes, the
smallest power of two covering the largest admitted element (the
7857 B SLH-DSA signature with an explicit sighash byte). This outer
cap is defense in depth only: the element size check runs before
execution, when the scheme is not yet known, so it has to be generic.
The exact size enforcement lives in the opcode itself (section 6),
which knows the scheme and rejects anything that is not the precise
signature and pubkey size for it. All other scripts keep the 520 B
rule untouched.

Rejected: chunking signatures into 520-byte pieces and reassembling
in the interpreter. It adds parsing complexity and per-element
overhead for zero benefit once the OP_SUCCESS ordering above is on
the table.

## 3. Leaf script form: commit to a hash, reveal the key at spend

The leaf script is 35 bytes for every scheme (1-byte push opcode, 33
bytes of data, 1-byte opcode):

    <scheme_byte(1) || H(pubkey)(32)> OP_CHECKPQSIG

H is a BIP 340 style tagged hash, tag "PQPubKeyHash" (name is
provisional). Domain separation is the convention everywhere post
taproot, the tag midstate is a precomputed constant so it costs
nothing, and a plain sha256 commitment would invite cross-protocol
collision arguments in review (the same 32 bytes could be a P2WSH
program, for one). Prototyping without the tag would mean breaking
every test vector later, since a final BIP would certainly add it.

The full public key goes in the witness, next to the signature.
Scheme ids reuse the libbitcoinpqc enum (1 = ML-DSA-44, 2 =
SLH-DSA-SHA2-128s; 0 is secp256k1 in the library and is not exposed
through this opcode).

Rejected: pushing the public key directly in the leaf script. The
ML-DSA key is 1312 B and does not fit a 520 B script push, so the
direct form could never be uniform across schemes. The hash form
also keeps the public key off chain until spend time, which is the
property P2MR exists to provide; publishing PQ pubkeys in advance
would give up part of it for no saving.

## 4. Witness layout for a PQ leaf spend

    [signature, pubkey, leaf script, control block] (+ annex per BIP 341)

The signature element is the raw scheme signature, with one trailing
sighash-type byte when the type is not SIGHASH_DEFAULT, the exact
convention BIP 341 uses for Schnorr (so 7856 or 7857 B for SLH-DSA,
2420 or 2421 B for ML-DSA-44). And the same fine print: an explicit
trailing 0x00 is invalid, because allowing it would give every
signature two valid encodings, the malleability BIP 341 closed for
Schnorr.

## 5. Sighash: BIP 342 unchanged

The message the PQ scheme signs is the same 32-byte tapscript sighash
P2MR already computes (SignatureHashSchnorr with the BIP 342
extensions; the midstate is already precomputed for P2MR spends).
Zero new sighash code, same hashtype rules.

Rejected: any custom message construction. There is no property to
gain at this layer, and P2MR's "no new sighash code" line is worth
keeping.

## 6. Opcode semantics

OP_CHECKPQSIG pops the 33-byte commitment (pushed by the script),
then the public key, then the signature.

- Commitment not exactly 33 bytes, or scheme byte unknown: fail the
  script. Future schemes arrive through a new OP_SUCCESSx or a
  defined upgrade of this one, not through lax parsing.
- Empty signature: push false and consume no budget, with only the
  commitment validated. The public key is deliberately not inspected
  on this path: in a branch construction over two schemes, the stack
  can carry a pubkey of the other scheme's size, and checking it here
  would break exactly the IF/ELSE case this rule exists to keep
  cheap. (Tapscript can hard-fail on empty pubkeys because all its
  keys are 32 bytes; that logic does not transfer to multi-scheme.)
- With a non-empty signature, public key and signature must have
  exactly the sizes the scheme defines (pubkey 1312 B and sig
  2420/2421 B for ML-DSA-44, pubkey 32 B and sig 7856/7857 B for
  SLH-DSA-SHA2-128s), else fail. Strict witness structure, same
  rationale as the P2MR rules: no third-party malleability surface.
- H(pubkey) must equal the committed hash, else fail.
- Any non-empty signature that does not verify: fail the script
  immediately, the BIP 342 rule that removes signature malleability
  as a relay vector.
- On success: push true.

One footgun to write down: the BIP 342 scan still short-circuits to
unconditional success on any OTHER OP_SUCCESSx in the same script, so
a leaf that wants PQ enforcement must not contain one. That is not a
soundness problem for the fork (the script author controls the
script), but wallet code building these leaves has to know it.

A side effect worth recording: because the opcode lives in ordinary
tapscript, hybrid scripts at the user level (a PQ check and an EC
OP_CHECKSIG in the same leaf) work with no extra spec. Whether a
final BIP wants to allow or forbid that is a spec question I am not
deciding here; the prototype allows it and can measure it.

## 7. Validation weight

Each passing OP_CHECKPQSIG subtracts a per-scheme constant from the
BIP 342 budget (serialized witness size + 50), calibrated as

    weight_scheme = 50 * ceil(median over runs of t_scheme / t_schnorr)

where t is the time one check takes: the public key parse and the
verification for Schnorr, the leaf commitment hash and the
verification for a PQ scheme. Each side gets the work its opcode
repeats per check and nothing else. The sighash is excluded from
both. That is not neutral, since adding a shared term to both sides
of a ratio above one pulls it toward one, so excluding it reads the
ratio high and the constant with it. The bench is
src/bench/pqc_verify.cpp.

Across five runs on my machine, ML-DSA-44 measures 2.99 times a
Schnorr check at the median and SLH-DSA-128s 14.36. The ratio I
apply is the highest of the runs rather than the median, 3.03 and
14.43, and the reason is ML-DSA: its ratio sits so close to 3 that
the runs fall on both sides of it, and median and maximum ceil to
different constants. When a ratio lands on an integer that way, the
safe reading is the one that charges more. The ceilings are 4 and
15, giving 200 for ML-DSA-44 and 750 for SLH-DSA-128s.

Neither margin is comfortable. ML-DSA would have to rise 32% to
reach the next constant up but only fall 1% to drop to 150, and
SLH-DSA has 4% above and 3% below. Both numbers can move on other
hardware, which is the argument for running this bench somewhere
else before any of it is treated as settled.

In seconds rather than units: the worst case the budget bounds is a
block packed with leaves that repeat one check, which at four
million witness bytes allows 80,000 Schnorr checks (4.42 s of
verification at the median), 20,000 ML-DSA checks (3.30 s) or 5,333
SLH-DSA checks (4.24 s). The three landing close together is the
formula working, since dividing by a cost proportional to the time
cancels the time. The direction is what the ceiling buys: both PQ
schemes come in under the Schnorr worst case the network already
accepts, where rounding SLH-DSA down to 700 instead would put it at
4.54 s, over it.

Both vendored trees are portable C with no SIMD, against a baseline
of an optimized libsecp256k1. The ML-DSA one is the reference
implementation, whose upstream ships an AVX2 variant I have not
measured; the SLH-DSA one is a portable C90 implementation that
offers no optimized variant at all, though faster SLH-DSA code
exists elsewhere. Either way these ratios bound this code rather
than the schemes, and a deployment against optimized
implementations needs its own calibration. These are also numbers
from one machine, and a BIP-grade calibration wants the same bench
run on more hardware.

The witnesses fund their own costs with room to spare. An SLH-DSA
spend carries a 7963-byte witness (budget 8013) and costs 750; an
ML-DSA spend carries 3809 bytes (budget 3859) and costs 200. The
budget only binds when a leaf repeats checks against the same
signature bytes: eleven SLH-DSA checks fit where the twelfth fails,
twenty-three ML-DSA checks fit where the twenty-fourth fails. Both
boundaries are pinned from both sides in script_tests.cpp and
through real blocks in feature_pqsig.py.

One boundary pair resolves a per-check cost only to about one part
in the number of checks the witness affords, which for SLH-DSA is
eight, so its pair alone accepts anything from 924 to 1034. A second
pair at a different witness size narrows that, since the two windows
intersect: padding the witness with one PQ_MAX_ELEMENT_SIZE element
buys budget without buying checks, affords sixteen, and leaves 989
through 1034 for the two together.

Sizing the padding deliberately also reaches the one case that
separates this rule from an off-by-one. BIP 342 fails a spend when
the budget goes negative rather than when it reaches zero, so a
spend that ends on exactly zero has to pass; 681 bytes of padding
funds 9000 units against nine checks costing 9000.

Rejected: costing a PQ verify at the flat 50 units of a Schnorr
sigop. There was never a reason to assume PQ verification lands at
Schnorr cost, and it does not; the measured ratios are 3 and 19.

## Open questions

- PQ_MAX_ELEMENT_SIZE: 8192 fits the two prototype schemes. A real
  BIP would size it to whatever scheme set it admits (FN-DSA and
  SHRINCS are smaller; SLH-DSA at higher security levels is much
  larger).
- The hybrid property from section 6 is measured: one leaf requiring
  an EC and an SLH-DSA signature together spends for 8063 witness
  bytes, 100 more than the PQ-only leaf. What stays open is whether a
  final BIP wants the hybrid form at all.
- Whether the scheme byte should encode security level variants
  (ML-DSA-65/87) or each variant gets its own id. Prototype: one id
  per exact parameter set, nothing implicit. Wiring the second scheme
  up showed the byte does more than select a verifier: since each
  scheme has fixed object sizes, it also fixes what the witness may
  carry, so a signature from the wrong scheme fails on size rather
  than reaching verification at all.
- key_version: the sighash reuses key_version 0, the value BIP 342
  assigns to BIP 340 keys, even though the field exists precisely for
  new key types. I think it is safe here (signature sizes are
  disjoint across schemes, so no signature verifies under two
  interpretations), but a final BIP should argue this properly or
  bump the version.
- Policy/standardness for relay of PQ spends: prototype runs with
  -acceptnonstdtxn=1 like the P2MR work did; a real deployment needs
  its own standardness rules. Note the asymmetry the flag placement
  creates: mempool policy applies the opcode on every network, while
  block validation applies it only on regtest, so on mainnet a 0xbb
  leaf stops being rejected by DISCOURAGE_OP_SUCCESS and executes
  instead. That direction is safe (mainnet consensus still treats
  0xbb as OP_SUCCESS, so nothing the mempool accepts is invalid), and
  real PQ spends stay non-standard anyway because the 80-byte
  tapscript stack item limit rejects a multi-kB signature. Narrowing
  the flag to regtest in the mempool would be worse: regtest would
  then accept into the mempool what regtest blocks reject.
- If the 2702 witness-style design lands, the encoding here migrates
  to that area; sizes in raw bytes carry over unchanged.
- The constants assume every check pays for a full verification,
  which is what makes charging per check the right shape at all. The
  block level aggregation of hash-based signatures raised on Delving
  2749 would price a signature by proving cost instead, and then it
  is the budget mechanism that has to change rather than its
  constant. Nothing follows from that here, since you cannot
  aggregate a verification nobody has specified yet, but this opcode
  is the non-aggregated base case and the premise should be on the
  page.
