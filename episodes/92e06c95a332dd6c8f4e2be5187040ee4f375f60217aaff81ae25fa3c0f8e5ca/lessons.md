# Lessons: anza-cryptography

Lineage `92e06c95a332dd6c8f4e2be5187040ee4f375f60217aaff81ae25fa3c0f8e5ca` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 1 published attempt.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:scalar_mul_ir

- Attempts: 1 (1 accepted)
- Best accepted effect: 4.86% (ratio 0.9514), episode `504766111cf3`
- Files changed by accepted candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.2537 USD (over 1 attempt with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I replaced the doubling formula in ProjectivePoint::double in secp256r1/src/group.rs, lines 359-364, 6 lines down to 5. The old code used the generic 1M+8S Jacobian doubling. The new code uses the a = -3 form (dbl-2001-b), 3M+5S. It computes alpha = 3(X - Z^2)(X + Z^2) and S = 4*X*Y^2 directly. X3, Y3 and Z3 come out as the same field values, so the coordinates match exactly.
> Sandbox run before replay: build ok on base and candidate, outputs identical. scalar_mul_ir went from 358282718 to 340880585, ratio 0.9514. msm_ir went from 280764894 to 273206234, ratio 0.9731. decode_ir stayed at 205933305, ratio 1.0000. Guards were within limits. I submitted it and it is queued for independent replay.
> Doubling is a large share of scalar_mul cost here, and the curve's a = -3 had not been exploited. decode does not touch this path.
> Next, look at the add and mixed-add paths in group.rs for the same kind of redundant squarings. After that, check whether field.rs square is cheaper than mul and used everywhere it applies.
>
> 2026-10-13, episode `504766111cf3` (accepted)

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
