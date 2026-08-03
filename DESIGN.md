# OP_CHECKPQSIG: design note

Design for the prototype on the `pq-opcode` branch. Every decision
lists the alternative I rejected and why. Nothing here is final spec;
it is the shape I am going to build and measure. Status: draft, not
yet implemented.

## 1. Delivery: redefine an OP_SUCCESSx inside tapscript

The opcode is a redefinition of OP_SUCCESS187 (0xbb) under leaf
version 0xc0, the same leaves P2MR already executes with BIP 342
rules. The prototype gates it behind a regtest-only script flag
(SCRIPT_VERIFY_PQSIG), the same pattern SCRIPT_VERIFY_P2MR uses.

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
- Public key and signature must have exactly the sizes the scheme
  defines (pubkey 1312 B and sig 2420/2421 B for ML-DSA-44, pubkey
  32 B and sig 7856/7857 B for SLH-DSA-SHA2-128s), else fail. Strict
  witness structure, same rationale as the P2MR rules: no third-party
  malleability surface.
- H(pubkey) must equal the committed hash, else fail.
- Empty signature: push false and consume no budget. Mirrors BIP 342,
  keeps branch constructions (IF/ELSE over two schemes) cheap.
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
BIP 342 budget (serialized witness size + 50). Placeholders until the
stage 5 bench calibrates them:

    cost_scheme = 50 * ceil(verify_time_scheme / verify_time_schnorr)

The large witnesses partly self-fund the budget (a 7.9 kB witness
brings a 7.9k budget), and one thing stage 5 must answer is whether
the calibrated constants exceed what the witness itself funds; if
they do, that is exactly the data the costing discussion (Delving
2702) needs.

Rejected: costing a PQ verify at the flat 50 units of a Schnorr
sigop. Consensus only pays for verification, and the stage 0 numbers
do not isolate it yet (the 1.2 s round trip is dominated by keygen
and signing), but there is no reason to assume PQ verification lands
at Schnorr cost, and undercosting validation CPU is the exact failure
mode the budget exists to prevent. The verify-only measurement is the
first number the stage 5 bench produces.

## Open questions

- PQ_MAX_ELEMENT_SIZE: 8192 fits the two prototype schemes. A real
  BIP would size it to whatever scheme set it admits (FN-DSA and
  SHRINCS are smaller; SLH-DSA at higher security levels is much
  larger).
- The hybrid property from section 6 deserves a measured row of its
  own: one leaf requiring both a PQ and an EC signature, end to end.
  Nobody has published that number.
- Whether the scheme byte should encode security level variants
  (ML-DSA-65/87) or each variant gets its own id. Prototype: one id
  per exact parameter set, nothing implicit.
- key_version: the sighash reuses key_version 0, the value BIP 342
  assigns to BIP 340 keys, even though the field exists precisely for
  new key types. I think it is safe here (signature sizes are
  disjoint across schemes, so no signature verifies under two
  interpretations), but a final BIP should argue this properly or
  bump the version.
- Policy/standardness for relay of PQ spends: prototype runs with
  -acceptnonstdtxn=1 like the P2MR work did; a real deployment needs
  its own standardness rules.
- If the 2702 witness-style design lands, the encoding here migrates
  to that area; sizes in raw bytes carry over unchanged.
