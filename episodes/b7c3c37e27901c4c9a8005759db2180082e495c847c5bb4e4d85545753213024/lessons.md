# Lessons: cryptography

Lineage `b7c3c37e27901c4c9a8005759db2180082e495c847c5bb4e4d85545753213024` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 10 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:scalar_arith_ir

- Attempts: 7 (4 accepted, 2 rejected, 1 no candidate)
- Best accepted effect: 44.19% (ratio 0.5581), episode `5c80d802b931`
- Accepted effects: 44.19%, 8.50%, 5.24%, 2.94%
- Why candidates failed: stale_conflict (1), dependency_failed (1)
- Files changed by accepted candidates: `secp256r1/src/scalar.rs`
- Files changed by rejected candidates: `secp256r1/src/scalar.rs`
- Median model spend per attempt: 0.3948 USD (over 7 attempts with a recorded cost)

## Target perf:field_arith_ir

- Attempts: 3 (2 accepted, 1 rejected)
- Best accepted effect: 3.07% (ratio 0.9693), episode `f44efdebcf29`
- Accepted effects: 3.07%, 1.73%
- Why candidates failed: stale (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`
- Files changed by rejected candidates: `secp256r1/src/field.rs`
- Median model spend per attempt: 0.5500 USD (over 3 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I changed the main loop of invert_canonical in secp256r1/src/field.rs, lines 288-299, which is the almost Montgomery inverse behind FieldElement::invert. The old loop checked both u and v for one on every iteration and negated (u - v) in the v < u branch. The new loop computes v - u directly with sub_limbs, checks only the value it just updated, and breaks when that value is one.
> My sandbox evaluation on field_arith_ir was accepted. field_arith_ir went from 373729639 to 367268003, ratio 0.9827. scalar_arith_ir was unchanged at 431938107, ratio 1.0000. Outputs were identical and the guard passed with 1 file and 29 lines. I submitted it, and it is queued for independent replay.
> The redundant one-checks and the extra negation were measurable overhead in the inverse. Only one side changes per iteration, so only that side needs a test.
> Next I want to look at the remaining field.rs functions that field_arith exercises and see whether the other inversion helpers have the same double check.
>
> 2026-10-13, episode `4426eddecf58` (accepted)

> I tried to cut instruction count in secp256r1/src/scalar.rs for scalar_arith_ir. First I rewrote Scalar montgomery_square as schoolbook squaring: off-diagonal macs, a doubling shift, then the diagonal terms. Then I replaced the multiplies by n's two top limbs (2^64-1 and 2^64-2^32) with shifts in the reduction. Last, I reworked lines 183-237.
> The squaring rewrite gave identical outputs and ratio 0.9964 (431929276 to 430389710). It was rejected against the required 0.9900. The shift change gave exactly the same count, ratio 0.9964 again. The last edit made things worse: 518532064, ratio 1.2005. field_arith_ir stayed at ratio 1.0000. I submitted nothing.
> The squaring gain is real but small. The compiler already lowers the top-limb multiplies to shifts or equivalent, so doing it by hand adds nothing. The 183-237 path is sensitive and easy to regress.
> Next: start from the schoolbook square and look for a second saving in the reduction loop or in mul, so the two together clear 1%.
>
> 2026-10-13, episode `878f35c9448f` (no candidate)

> Changed Scalar::invert in secp256r1/src/scalar.rs. The old version converted out of Montgomery form, ran a binary extended Euclidean inversion that halved a cofactor modulo n for every removed bit (an add of n plus a shift each time), then multiplied by R2. It now uses the almost Montgomery inverse (Kaliski) on the Montgomery limbs. FieldElement::invert in field.rs already uses that algorithm. Two edits, lines 107-111 and 170-236, 142 changed lines.
> Sandbox evaluation 1 was accepted. scalar_arith_ir went from 472193510 to 453720658, ratio 0.9609. field_arith_ir went from 359051774 to 359073631, ratio 1.0001, within its guard limit. Outputs were identical and both builds were ok. This is my own measurement. The verdict comes from replays later.
> Most of the cost was the per-bit halving modulo n. Doubling the cofactors does less work per step.
> Next, profile what remains in Scalar::invert and check whether the final correction multiply can be merged with another step.
>
> 2026-10-13, episode `0ec3893b4855` (rejected)

> I rewrote montgomery_square in secp256r1/src/field.rs. The old version built the 512-bit square with u128 overflowing adds and tracked carries in every column. The new one is schoolbook squaring: six off-diagonal products in mac chains, a shift to double them, then four diagonal products added with adc. I also edited lines 277-312 and line 81 of the same file.
> The first evaluation was rejected. field_arith_ir went from 374713833 to 373446221, ratio 0.9966, and the required ratio was 0.9900. The third evaluation was accepted at 361716489, ratio 0.9653, with 135 changed lines. scalar_arith_ir held at 846026188 in both runs. Outputs were identical both times. It is submitted and queued for replay.
> The squaring rewrite alone barely moved field_arith_ir. Most of the gain came from the later edits around lines 277-312.
> Next I want to isolate which of those edits carries the 0.9653. After that I will check whether montgomery_mul has the same per-column carry pattern.
>
> 2026-10-13, episode `63c0f31858b2` (rejected)

> I changed Scalar::invert in secp256r1/src/scalar.rs. The old path converted out of Montgomery form, ran the binary extended Euclidean inversion, then multiplied by R2. invert_canonical now takes a seed for the first cofactor and returns seed * a^-1 mod n. Feeding it the Montgomery limbs (aR) with seed R2 gives a^-1 R directly. That drops one Montgomery reduction and one Montgomery multiplication. I also restructured the loop. The edits were lines 107-111 and 172-207.
> Sandbox evaluation 1: scalar_arith_ir went from 472189588 to 431932649, ratio 0.9147. field_arith_ir went from 385713616 to 385719604, ratio 1.0000. Outputs were identical and the guard passed. I submitted it, and it is waiting on replays.
> Scalar inversion carried a redundant round trip through Montgomery form. The seeded cofactor removes it with no change to the field side.
> Next I want to check whether field.rs inversion has the same convert, invert, multiply-by-R2 pattern. I also want to see if the seeded form applies there.
>
> 2026-10-13, episode `437eb3bffef9` (accepted)

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
