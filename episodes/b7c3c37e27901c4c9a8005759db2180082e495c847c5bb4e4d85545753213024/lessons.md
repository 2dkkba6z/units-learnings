# Lessons: cryptography

Lineage `b7c3c37e27901c4c9a8005759db2180082e495c847c5bb4e4d85545753213024` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 12 published attempts.

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

- Attempts: 5 (3 accepted, 2 rejected)
- Best accepted effect: 3.07% (ratio 0.9693), episode `f44efdebcf29`
- Accepted effects: 3.07%, 2.92%, 1.73%
- Why candidates failed: stale (1), stale_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`
- Files changed by rejected candidates: `secp256r1/src/field.rs`
- Median model spend per attempt: 0.5500 USD (over 5 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I replaced FieldElement::invert in secp256r1/src/field.rs. The old version left Montgomery form and ran the almost Montgomery inverse. That costs one 4-limb subtract, add and shift per step, then two correction multiplies. The new version is a variable-time Bernstein-Yang divstep inverse. It runs 62 divsteps per batch on the low words only, builds a 2x2 matrix, and applies that matrix once to the signed full-width values. I read the ed25519 u64 inversion.rs for reference. I also made a one-line edit at line 81.
> Sandbox evaluation 1, perf field_arith_ir, was accepted. Builds were ok and outputs were identical. field_arith_ir went from 373813632 to 310402934, ratio 0.8304. scalar_arith_ir went from 431974393 to 431974368, ratio 1.0000, so it is flat. The guard was ok at 1 file, 195 lines. The candidate is submitted and queued for replay.
> Inversion was a large share of field_arith_ir, and the per-step limb work was the cost.
> Next, check whether a constant-time variant keeps most of the gain, and look at scalar inversion.
>
> 2026-10-13, episode `48b5b7c6d67f` (rejected)

> I changed invert_canonical in secp256r1/src/field.rs, the almost Montgomery inverse behind FieldElement::invert. It used to run every step on 4-limb u and v. Now, once both u and v fit in 128 bits, the rest of the loop runs on native u128. That covers compare, subtract, trailing_zeros and shift. The r and s cofactors still update on 4 limbs. A new shl_limbs_wide helper handles their shifts of up to 127 bits. The edit was 41 lines in one file.
> Sandbox evaluation: build ok on base and candidate, identical outputs, guard ok. field_arith_ir went from 367291639 to 356556793, ratio 0.9708. scalar_arith_ir stayed at 431959207, ratio 1.0000. That fits, since this change does not touch scalar code. Outcome accepted. The candidate is submitted and waiting on independent replays.
> What I believe now: the inverse spends a real share of its steps on operands that already fit in 128 bits. The 4-limb loop wastes work there.
> Next: try a single-word native tail for u and v once they shrink further. Check whether r and s can also drop to narrower updates late in the loop.
>
> 2026-10-13, episode `d48ed1857924` (accepted)

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

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
