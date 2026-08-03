# PQ opcode: working notes

The goal is to prototype the "TBD Post Quantum Signature BIP" that
BIP 361 requires and BIP 360 leaves open (line 338: the authors are
"currently researching options"). Code goes on branch `pq-opcode`,
stacked on `p2mr-regtest` in my Core fork. This repo is the journal,
same format as bitcoin-p2mr.

The stages I am working through, referred to by number below: 0 recon
and library choice, 1 design note, 2 first scheme end to end, 3 second
scheme and the scheme dispatch, 4 functional test through a real
block, 5 measurements.

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
- Weight: the validation weight budget test I wrote for P2MR
  (bitcoin-p2mr, FUTURE item 8) is already on this branch, since it
  is stacked on p2mr-regtest. It is the harness the measurements
  extend.

The three questions above that stage 0 and stage 1 answered: the
library turned out to carry no sighash logic at all, the 520-byte
question is settled in DESIGN.md (redefine an OP_SUCCESS, lift the
bound for leaves that use it), and the SLH-DSA variant is SHA2-128s.

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

## Stage 2, 2026-08-03: SLH-DSA verify path working end to end

OP_CHECKPQSIG is implemented on the pq-opcode branch, following the
design note. What went in:

- src/pqc/: the sha2-128s subset of the SPHINCS+ reference code
  (provenance in its README), a scheme-agnostic wrapper
  (pqc_verify.{h,cpp}) and a bitcoin_pqc CMake target linked into
  bitcoin_consensus and the kernel. Consensus code only ever calls
  Verify(); keygen and signing exist for tests.
- Interpreter: OP_CHECKPQSIG = 0xbb in the opcode table, the
  SCRIPT_VERIFY_PQSIG flag (regtest block validation only, same
  wiring as SCRIPT_VERIFY_P2MR), the OP_SUCCESSx scan exception, the
  PQ_MAX_ELEMENT_SIZE = 8192 initial-stack bound scoped to leaves
  that contain the opcode, EvalCheckPQSig mirroring
  EvalChecksigTapscript (empty sig pushes false with no budget cost,
  everything else fails hard), and CheckPQSignature on the checker
  computing the exact BIP 342 sighash the P2MR path already
  precomputes. Provisional budget cost: 1000 per passing check,
  stage 5 calibrates.
- Five new script errors (scheme, size, hashtype, pubkeyhash, and a
  bad-signature catch-all).

Tests: script_pqsig_slh_dsa in script_tests.cpp covers the valid
spend under SIGHASH_DEFAULT and SIGHASH_ALL, the soft-fork base case
(flag off means anyone-can-spend), and one negative per design rule:
bit-flipped sig, sig over the wrong sighash, explicit 0x00 sighash
byte, out-of-range 0x04 sighash byte, sig one byte short and one byte
long, wrong pubkey size, right size but wrong key, empty sig (fails
as EVAL_FALSE, not an abort), unknown scheme byte, 32-byte
commitment, missing stack element, oversized element in a PQ leaf,
521-byte element in a non-PQ leaf (the old limit stays), and the
other-OP_SUCCESSx short-circuit from the design note. There is also a
hybrid leaf requiring both an SLH-DSA and a Schnorr signature over
the same sighash, which I have not seen demonstrated in code
anywhere.

Three cases pin behavior that is easy to get wrong later: an ML-DSA
leaf is unspendable (not anyone-can-spend) until stage 3 wires the
scheme up; the larger element bound follows the textual presence of
the opcode, not its execution, since the scan runs before any branch
is evaluated (an unexecuted OP_CHECKPQSIG inside an IF still lifts
it); and the same PQ leaf spends identically under a BIP 341 taproot
script path, so a P2TRv2-style output reusing this opcode needs no
separate verification code.

Full suite: 801 unit test cases green, feature_p2mr.py green,
p2mr_vector_tests green.

Mutation testing (break each rule on purpose, see which tests notice),
the same method I used on the P2MR stages: the commitment size check,
the scheme check, the pubkey size check, the explicit-0x00 sighash
rejection, the empty-signature short circuit, the flag gate in the
scan and the element-limit scoping all get caught. Removing the
budget subtraction did not, because one 7857-byte signature funds far
more budget than a single check spends. A leaf that repeats the check
over the same signature (OP_2DUP, commitment, OP_CHECKPQSIG,
OP_VERIFY) funds no extra budget, so eight repetitions pass and nine
run out; that test now kills the mutant. Adding it also filled a gap
in Core's own table of script error names, which had no entry for
TAPSCRIPT_VALIDATION_WEIGHT.

feature_taproot.py needed a one-line change. That test loops over
every OP_SUCCESSx value and asserts the unconditional-success
behavior (bare success, big-push bypass, and so on). Because regtest
block validation now applies SCRIPT_VERIFY_PQSIG, 0xbb no longer
succeeds unconditionally and the loop has to skip it. The edit is
trivial, the implication less so: redefining an OP_SUCCESS in place
means every consumer that treats the range uniformly has to learn the
exception. A real BIP could sidestep it by picking an OP_SUCCESS
number nothing special-cases, or by going through the pqdata witness
area (2702) instead. Recorded in the design note as input to that
discussion.

A few decisions and warts worth recording:

- PQ verification goes through a VerifyPQSignature virtual on the
  checker, the same seam VerifySchnorrSignature uses, because that is
  where CachingTransactionSignatureChecker hooks the signature cache.
  PQ verify is the most expensive check in the interpreter by far, so
  it will need the cache before it means anything outside regtest;
  the seam is there, the cache wiring is not.
- The budget constant applies to any scheme for now, so it is named
  VALIDATION_WEIGHT_PQSIG without a scheme suffix; stage 5 splits it
  per scheme when the calibration numbers exist.
- MAX_OPCODE in script.h is OP_NOP10 (0xb9), so OP_CHECKSIGADD (0xba)
  already sits outside HasValidOps and the asm parser upstream.
  OP_CHECKPQSIG (0xbb) gets the identical treatment and I left it
  alone: none of those paths see tapscript leaves anyway, since they
  check scriptPubKey and scriptSig while the leaf lives in the
  witness.
- The vendored sign.c drags randombytes into the consensus library as
  dead code (verification never calls it). A real PR would split the
  sign path out of the consensus target; for the prototype it stays,
  recorded here so it is a known wart and not an accident.
- SCRIPT_VERIFY_PQSIG sits in MANDATORY_SCRIPT_VERIFY_FLAGS, which
  mempool policy applies on every network, while GetBlockScriptFlags
  adds it only on regtest. So on mainnet a tapscript leaf containing
  0xbb is no longer turned away by DISCOURAGE_OP_SUCCESS; it runs the
  opcode. Nothing unsafe follows from that (mainnet consensus still
  reads 0xbb as OP_SUCCESS, so every transaction the mempool accepts
  is consensus-valid), and any real PQ spend is still nonstandard
  under the 80-byte item limit. The alternative, keeping the flag out
  of the mempool, would have regtest accepting transactions its own
  blocks reject, which is the direction that actually causes trouble.
  P2MR carries the same structure for the same reason.
- Relay policy caps tapscript stack items at 80 bytes
  (MAX_STANDARD_TAPSCRIPT_STACK_ITEM_SIZE, and the P2MR policy
  mirrors it), so every PQ spend is nonstandard today. Block
  validation is unaffected; the functional test has to submit blocks
  directly or run with -acceptnonstdtxn=1. Real standardness rules
  are a spec question the design note already lists.
- The default build config does not build the bitcoinkernel target,
  so the pqc wiring in src/kernel/CMakeLists.txt sat unexercised
  until I built it explicitly with BUILD_KERNEL_LIB=ON in a throwaway
  build dir: compiles, links, and the pqc symbols land in
  libbitcoinkernel.a. The vendored files are byte-identical to the
  pinned libbitcoinpqc checkout (cmp over every file).
- The empty-signature path validates the commitment and nothing else:
  with different pubkey sizes per scheme, inspecting the pubkey there
  would break IF/ELSE constructions over two schemes. The design note
  spells this out and a test pins it.

## Next steps

1. Stage 3: wire up ML-DSA-44 as scheme byte 1 and exercise the
   scheme dispatch with cross-scheme negatives.
2. Stage 4: feature_pqsig.py, spending through the PQ leaf in a real
   block and rejecting a tampered one via submitblock.
3. Stage 5: calibrate the budget constant from measured verify time;
   the comparison table including paper numbers for FN-DSA, SHRINCS
   and SQIsign.
