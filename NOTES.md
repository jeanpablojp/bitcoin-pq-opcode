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
budget subtraction did not, because one 7856-byte signature funds far
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
- The signing half of the vendored code is compiled into the
  consensus library even though verification is all consensus ever
  calls. A real PR would split it out of that target; for the
  prototype it stays, recorded here so it is a known wart and not an
  accident.
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

## Stage 3, 2026-08-04: ML-DSA-44 on scheme byte 1

Both schemes now verify under the same opcode. The scheme-agnostic
design survived contact with a second implementation, but not for
free:

- The two reference implementations disagree about symbol naming.
  Dilithium's sign.h macro-renames crypto_sign_* onto
  pqcrystals_dilithium2_ref_*, while SPHINCS+ keeps the plain names.
  Including both headers in one translation unit would let the
  Dilithium macros rewrite calls meant for SPHINCS+, silently. Each
  scheme now sits behind its own file (slh_dsa.cpp, ml_dsa.cpp) with
  a plain C++ interface, and pqc_verify.cpp dispatches without
  including either crypto header. I checked the object files rather
  than trusting the build: ml_dsa.cpp.o references only
  pqcrystals_dilithium2_ref_*, slh_dsa.cpp.o only crypto_sign_*.
- Both trees ship a randombytes.c and both define randombytes(), so
  vendoring either would eventually collide. Neither is vendored now.
  randombytes.cpp provides one definition instead, seeded through
  SeedKeypair and aborting if nothing was installed. Consensus never
  reaches it: verification draws no randomness. That guard is for
  vendored code drawing randomness nobody arranged for, not for
  ordinary misuse, so SeedKeypair rejects an empty seed and Sign
  returns false when no entropy is installed. Both used to abort the
  process, which is not what a bad argument deserves.

  Where each scheme actually needs it is worth writing down, because
  the headers mislead on this. Dilithium draws
  in key generation, which has no seeded entry point upstream, unlike
  SPHINCS+ which takes a seed directly. Both also draw while signing:
  SPHINCS+ randomizes its message digest, and this Dilithium copy
  ships config.h with DILITHIUM_RANDOMIZED_SIGNING defined, so the
  branch that compiles is not the deterministic one the #ifdef shows
  first. Neither scheme signs deterministically on its own; both are
  reproducible from a reinstalled seed, and a test pins that.
- The size checks turned out to double as scheme separation. An
  SLH-DSA key and signature offered against a leaf committing to
  ML-DSA fails on size before anything else, and a leaf committing to
  the right key under the wrong scheme byte fails on the commitment.
  Both are tested.

Tests: script_pqsig_ml_dsa covers a valid ML-DSA spend, a bit-flipped
signature, the two cross-scheme cases above, and one leaf that
requires both schemes with a key each. The stage 2 case asserting an
ML-DSA leaf was unspendable is gone, replaced by that valid spend.
Reproducibility is asserted rather than assumed: the same seed gives
the same key pair, and two signatures over the same message match
when the seed is reinstalled between them, which is the property
future vectors would rely on.

Mutation testing on the dispatch caught swapping which backend a
scheme reaches and swapping the two schemes' declared sizes. Dropping
the size check inside Verify() survived, and the reason is structural:
the opcode enforces exact sizes before calling, so nothing reached the
wrapper's own guard. That guard is the wrapper's contract rather than
dead weight, since it is what makes the function safe to call from
anywhere else, so it now has a test of its own (pqsig_verify_sizes)
that exercises Verify directly with every wrong size, the other
scheme's sizes, and unknown scheme bytes.

The entropy hook needed a test of its own for the same kind of
reason. Removing the counter increment from randombytes.cpp, which
makes it hand back the same block over and over, changed nothing any
test could see: every caller today asks for at most one block. A
vector generator asking for more would have got correlated keys and
no warning. pqsig_entropy_stream now checks the stream directly, that
a long draw has distinct blocks, that successive draws continue
rather than restart, that reinstalling the seed rewinds, and that a
different seed gives a different stream.

Mutating the per-scheme backends is caught as well: a message length
of 31 instead of 32, in either direction, and discarding the verify
return value.

Both schemes also run clean under AddressSanitizer and
UndefinedBehaviorSanitizer, across key generation, signing,
verification, the rejection paths, and 600 malformed
correctly-sized signatures fed to verification, which is where the
vendored parsers do their work. None were accepted.

Everything above only proves the code agrees with itself: the tests
sign and verify with the same implementation, so a wrong parameter
set or a botched call convention would pass unnoticed. No NIST
known-answer vectors ship with either tree, only the generators. What
I could do instead was cross-validate against a separately built
libbitcoinpqc: signatures produced by the vendored code here are
accepted by its bitcoin_pqc_verify for both schemes. That pins the
parameter sets, the encodings and the ML-DSA context argument, which
the two implementations pass differently in form (a zero-length
buffer against a null pointer) but identically in effect. Real
conformance still wants FIPS 204/205 vectors, and a companion BIP
would need them.

Full suite: 804 unit test cases green, feature_p2mr.py green,
feature_taproot.py green, and the kernel target still links with the
crypto dependency the entropy hook adds.

Two notes for anyone pruning the vendored trees. dilithium/api.h is
not reachable from anything we compile, so it is not vendored.
sphincsplus/sha2_offsets.h looks equally unused by name, but the
parameter header pulls it in through a relative path, so it stays;
grepping for the file name alone will tell you the opposite.

## Stage 4, 2026-08-05: spending through real blocks

feature_pqsig.py spends a PQ leaf under both schemes through a block
built here and handed to submitblock, and watches the whole block go
away when the signature is tampered with or the public key misses the
commitment. That is consensus deciding, which no interpreter test
reaches. Two constructions only a block can settle also pass: a leaf
requiring an SLH-DSA and a Schnorr signature over the same sighash,
and a leaf requiring both PQ schemes with a key each.

Python cannot produce a post-quantum signature, so this needed a
decision about how much of the test the node would do. The valuable
part of feature_p2mr.py was that Python built its own sighashes: an
independent reading of BIP 341/342 sitting across from the C++ one.
That still holds here. Python builds the sighash, the witness and the
block; two hidden regtest RPCs (pqpubkey, pqsignhash) supply only the
primitive Python has no way to compute. They keep nothing: the seed
arrives with each call, as the P2MR RPCs do.

The policy reasoning from stage 2 turned out right by a more precise
route than I described. I had expected the mempool to reject these on
script verification; it rejects them as bad-witness-nonstandard, so
the spend never reaches verification at all. The 80-byte standard
stack item limit turns it away on witness shape, and the same
transaction then goes into a block without complaint. The test asserts
both halves.

Mutation testing on the opcode, this time asking whether the
functional test would notice: discarding the signature result is
caught, and so is dropping the commitment check. The second one needs
the right witness to catch it. A public key with a flipped byte
proves nothing, because the signature check rejects that witness
whether or not the commitment was consulted. What isolates the
commitment is a signature that verifies perfectly, under a key the
spender chose, against a leaf committing to somebody else's: the
theft case, where the commitment is the only thing between an
attacker and another person's output. Both schemes have that test.

The RPCs are documented as keeping nothing, and the entropy state
made that untrue. Key generation installs the seed in a global that
outlives the call, so a caller's seed sat in process memory until the
next one arrived. ClearDeterministicEntropy wipes it from a
destructor, which covers the error paths too.

Wiring that global to an RPC also broke an assumption I had written
into it. The comment said nothing touching the entropy state runs on
more than one thread, which held while only tests reached it and
stopped holding the moment a handler did: the RPC server answers from
a thread pool. Four signatures fired at once came back in the time
one takes, so the overlap is real, and the sequence that has to be
atomic is install, generate, rewind, sign, wipe. Locking each access
would not do: another thread reinstalling between the install and the
draw changes the result, and one wiping there trips the guard and
takes the node down. EntropyLock holds a mutex across the whole
sequence, after which six concurrent signatures take 7.14 s against
7.19 s for the same six run one after another.

No wrong signature ever came out of it in testing, since the window
between installing entropy and consuming it is microseconds wide, so
timing was never going to settle whether the code was correct.
ThreadSanitizer does settle it: four threads through the sequence
report data races on g_seed without the lock and nothing with it,
and g_seed is the only location it names, so the diagnosis missed
nothing.

Verification deserved the same question, and more urgently, since
block validation runs script checks on several threads at once and a
shared byte there would be a consensus race rather than a test-tool
one. Six threads verifying at the same time across both schemes come
back clean under ThreadSanitizer. The vendored verify code keeps its
state on the stack, which is what lets the opcode run in a real
block.

Seeds were unbounded, which the P2MR RPCs in the same fork are not:
they cap leaf counts and sizes so a request cannot make the node
allocate at will. An 8 MB seed here was accepted and hashed on every
draw, all of it while holding the entropy lock. A seed is only
entropy and no scheme here draws more than 48 bytes of it, so it is
now bounded at both ends: exactly 48 for SLH-DSA, 32 to 64 for
ML-DSA, the lower bound so a one-byte seed cannot quietly produce a
weak key.

The run takes about fourteen seconds, nearly all of it SLH-DSA
signing at roughly 1.2 s a signature against an ML-DSA one that is
close to free. That gap is the first hint of what the measurements
will have to put numbers on, though signing is not the cost consensus
pays: verification is, and it is the one still unmeasured.

## Stage 5, 2026-08-06: measurements

The constant the interpreter charges per passing OP_CHECKPQSIG was a
placeholder until now. The design note fixed the shape of the answer
in advance, 50 units times a ratio against Schnorr rounded up to a
whole multiple, and left this stage to produce the ratio. Settling
what belongs inside that ratio is most of what follows.

The obvious comparison, the raw primitives of pqc::Verify against
VerifySchnorr, is not symmetric. VerifySchnorr parses the public key
before verifying, 5.7 µs of the 56 the call takes, and there is no
leaving that out because it happens inside the function consensus
calls. The PQ counterpart is the tagged hash of the public key that
the leaf commitment check needs, 5.4 µs for ML-DSA's 1312-byte key,
and a bench of the primitive alone misses it. Pricing one side with
its per-check overhead and not the other moves ML-DSA's ratio by 3%,
which would not matter if that ratio were not sitting a couple of
percent from an integer, with the constant depending on which side of
it the ratio falls.

The bench therefore measures what each opcode repeats per check:
public key parse plus verification for Schnorr, commitment hash plus
verification for the PQ schemes. The public key hash belongs in the
per-check cost even though the key is already paid for by the
witness size term, because a leaf repeating the check carries the
key once and hashes it once per check. That work scales with checks,
not with witness bytes, which is the case the constant has to cover.

Both benches leave out the sighash, which each path computes once
per check. That is not the neutral choice it looks like: adding the
same term to both sides of a ratio above one pulls the ratio toward
one, so at a sighash of 5 µs the SLH-DSA ratio would read 17.7
rather than 19.2 and the constant would come out at 900. Excluding
it reads the ratio high, which is the direction I want. It also
widens the block level margin below, since a Schnorr check pays for
a sighash twenty times as often as an SLH-DSA check does.

Sixteen runs on a laptop that also runs an editor and an antivirus,
with the reference C implementations, reported as median and the
spread across runs:

| per check              | median   | across runs  | ratio |
|------------------------|----------|--------------|-------|
| Schnorr (libsecp256k1) | 55.9 µs  | 54.8-58.0 µs | 1     |
| ML-DSA-44              | 171 µs   | 169-176 µs   | 3.07  |
| SLH-DSA-SHA2-128s      | 1074 µs  | 1065-1093 µs | 19.24 |

Sixteen because eleven were not enough: every batch I added widened
the min and max, which is what extremes of a sample do as long as
the noise keeps going. Within a run nanobench reports 0.2% to 1.2%
error; the spread between runs is the machine. Verification here
does a fixed amount of work, so that spread is measurement noise
rather than variation in what a check costs, and the median is the
estimate. The range is there to show the noise, not to bound the
cost; it would keep widening if I kept running.

The medians give ratios of 3.07 and 19.24, whose ceilings are 4 and
20, so 200 for ML-DSA-44 and 1000 for SLH-DSA-SHA2-128s. The run
where each came out worst against the baseline gives 3.20 and 19.46,
which ceil the same way. The constants therefore do not depend on
picking a central estimate over a pessimistic one, and that agreement
is worth more than either number on its own. The placeholder was 1000
for both, which happens to be the SLH-DSA answer.

Neither ratio has much room, though, and they have different amounts
of it. ML-DSA's median would have to climb 30% before the constant
moved to 250, while SLH-DSA's needs only 4% to reach 20.0 and push
its constant to 1050. So the SLH-DSA number is the one that a bench
run on faster or slower hardware is most likely to unseat, and the
one a real calibration should measure most carefully.

For the record, a series taken right after a compile, with the load
average at 27, gave ratios of 2.75-3.19 and 17.65-19.42. Those are
not in the sixteen; they measure the machine, not the code.

Units are hard to argue about. In seconds: the worst case the budget
bounds is a block packed with leaves that repeat one check, and a
block holds at most four million witness bytes, which puts the check
count at four million over the per-check cost.

At the medians above:

| scheme            | max checks | verification | vs Schnorr |
|-------------------|------------|--------------|------------|
| Schnorr           | 80,000     | 4.47 s       | 1.00       |
| ML-DSA-44         | 20,000     | 3.43 s       | 0.77       |
| SLH-DSA-SHA2-128s | 4,000      | 4.30 s       | 0.96       |

That the three land close together is the formula working rather
than an independent result, since dividing by a cost proportional to
the time cancels the time. What is not automatic is the direction:
both PQ schemes come in under the Schnorr worst case the network
already accepts, and they do so because the ceiling rounded both
costs up. Rounding down instead, which the fastest SLH-DSA run would
have permitted at 950, allows 4210 checks and puts it at 4.52 s,
over the line. Taking the ceiling is what keeps it under.

Some caveats belong next to those numbers. Both vendored trees are
the reference implementations, portable C with no SIMD anywhere in
them, and Dilithium's symbols say so outright:
pqcrystals_dilithium2_ref_*. The baseline they are measured against
is an optimized libsecp256k1. Upstream ships AVX2 variants of both
schemes that I have not built or measured, so the honest statement
is that these ratios bound this code rather than the schemes, and a
deployment against optimized implementations needs its own
calibration. And this is one machine. A BIP-grade calibration wants
the same bench run on more hardware than mine.

Splitting the shared constant in two touched script.h and one line
of the interpreter, where the subtraction goes through a switch on
the scheme with no default case, so adding a third scheme without
pricing it is a compiler warning rather than a silent inheritance of
SLH-DSA's cost. PubKeySize and SigSize are written the same way.

A leaf that repeats the check (OP_2DUP <commitment> OP_CHECKPQSIG
OP_VERIFY per repetition) adds 37 bytes of leaf script per check,
which buys 37 more units of budget against a check costing 200 or
1000, so repetition runs the budget out. I worked the boundaries out
on paper first: one SLH-DSA witness funds eight checks and fails on
the ninth, one ML-DSA witness funds twenty-three and fails on the
twenty-fourth. Both pairs then passed unchanged in script_tests.cpp
and through real blocks in feature_pqsig.py. The numbers in the
tests are predictions that held, not values copied from a first run.

A boundary pair pins a constant less tightly than it looks. Working
the inequalities backwards, the ML-DSA pair passes for any cost in
(196.5, 203.4], a fine 2%, but the SLH-DSA pair passes for anything
in (923.9, 1034.8]. Setting the constant to 950 and running the
suite confirms it: everything still passes. The resolution of a
boundary pair is about one part in n, where n is the number of
checks the witness affords, and n is eight for SLH-DSA precisely
because its checks are expensive.

The fix is a second boundary at a different witness size, since two
windows intersect to something narrower than either. Padding the
witness with one more element buys budget without buying checks, and
the largest element the leaf admits is already named:
PQ_MAX_ELEMENT_SIZE. With 8192 bytes of padding the same witness
affords sixteen checks rather than eight, giving (988.6, 1048.1],
and the two pairs together leave (988.6, 1034.8]. That is a 4.6%
window, and more to the point it excludes 950, which the padded pair
rejects.

Being able to size the padding to order turned out to be worth more
than the narrower window. BIP 342 fails a spend when the budget goes
negative, not when it hits zero, and no boundary case here lands on
zero: the tightest leaves 79 units. Choosing the padding so the
budget comes out exact reaches it. At 681 bytes the witness funds
9000 against nine checks costing 9000, so the spend has to pass with
nothing left, and changing the interpreter's test from negative to
non-positive breaks it. That off-by-one is invisible everywhere
except at zero, which is why it takes a case that lands there.

One reason the budget matters more here than it does for Schnorr:
the repeated-check leaf verifies the same signature over the same
sighash every time, and in block validation, where the signature
cache is storing, every repetition after the first is a cache hit on
the Schnorr side. CachingTransactionSignatureChecker overrides
VerifySchnorrSignature and not VerifyPQSignature, which nothing
wires up yet, so a PQ repetition pays in full. BIP 342 charges 50 a
check either way, but on the PQ side the budget is currently the
only thing bounding that case.

The size table, every row spent through a real block and the witness
sizes asserted byte for byte in feature_pqsig.py:

| spend                           | witness | weight | vsize | cost |
|---------------------------------|---------|--------|-------|------|
| P2TR key path                   | 66 B    | 444    | 111   | n/a  |
| P2TR script path, checksig leaf | 167 B   | 545    | 137   | 50   |
| P2MR, same checksig leaf        | 135 B   | 513    | 129   | 50   |
| P2MR, ML-DSA-44 leaf            | 3809 B  | 4187   | 1047  | 200  |
| P2MR, SLH-DSA-128s leaf         | 7963 B  | 8341   | 2086  | 1000 |
| P2MR, hybrid EC+SLH-DSA leaf    | 8063 B  | 8441   | 2111  | 1050 |

Weight and vsize are for the whole one-input, one-output test
transaction; the witness column is the per-input cost and is exact.
The cost column is the validation weight the leaf spends, which is a
tapscript rule: a key path spend runs no script and has no budget,
hence the n/a rather than a 50. The hybrid leaf pays for both of its
checks, 1000 and 50.

The 32-byte P2MR discount against the taproot script path is still
there in the PQ rows, but next to signatures in the thousands of
bytes it barely registers.

The hybrid row is a number I had not seen published anywhere: one
leaf requiring an EC and an SLH-DSA signature together spends for
8063 witness bytes, 100 more than the PQ-only leaf (65 for the
Schnorr signature with its prefix, 35 for the extra leaf bytes). A
hybrid spend costs 1.3% more than the PQ spend it wraps.

The self-funding question from the design note has its answer: both
schemes fund their own verification with room to spare. The SLH-DSA
witness brings a budget of 8013 against a cost of 1000, the ML-DSA
witness 3859 against 200. For single-signature leaves the calibrated
costs change nothing about what is spendable; they bind when a
script repeats checks against the same signature bytes, which is the
adversarial case they exist for.

For the schemes I am not implementing, paper numbers, with a witness
estimate computed from the same witness shape (signature, pubkey,
35-byte leaf, 33-byte control block, plus prefixes):

| scheme     | sig        | pubkey | est. witness      | status     |
|------------|------------|--------|-------------------|------------|
| FN-DSA-512 | ~666 B     | ~897 B | ~1.6 kB           | FIPS 206 draft |
| SHRINCS    | ~2.5 kB    | n/a    | ~2.6 kB + pubkey  | eprint 2025/2203 |
| SQIsign    | ~0.2 kB    | ~64 B  | ~0.4 kB           | NIST additional round |

Estimates, not measurements; nothing was built for these. The two
measured rows show why size alone does not rank the table: ML-DSA
and SLH-DSA differ by a factor of six in verify time, and the budget
constant has to be calibrated per scheme either way. SQIsign trades
its small signatures for verification cost, so its row would need
its own measurement before the size means much.

## Next steps

1. Spend vectors for the opcode, reusing the machinery
   feature_pqsig.py built, the same route the P2MR spend vectors
   took.
2. The write-up, once the vectors exist.
