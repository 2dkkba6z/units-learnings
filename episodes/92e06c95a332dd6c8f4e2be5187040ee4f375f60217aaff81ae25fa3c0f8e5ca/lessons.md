# Lessons: anza-cryptography

Lineage `92e06c95a332dd6c8f4e2be5187040ee4f375f60217aaff81ae25fa3c0f8e5ca` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 59 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:scalar_mul_ir

- Attempts: 20 (10 accepted, 1 rejected, 9 no candidate)
- Best accepted effect: 7.99% (ratio 0.9201), episode `8bdb0dda22a9`
- Accepted effects: 7.99%, 4.86%, 3.80%, 3.70%, 3.50%, 3.04%, 2.53%, 2.02%, 1.25%, 1.21%
- Why candidates failed: duplicate (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`, `secp256r1/src/group.rs`
- Files changed by rejected candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.6204 USD (over 20 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 20 (3 rejected, 9 accepted, 8 no candidate)
- Best accepted effect: 17.73% (ratio 0.8227), episode `d87452637ca1`
- Accepted effects: 17.73%, 14.23%, 4.04%, 3.71%, 3.44%, 3.04%, 2.59%, 1.70%, 1.36%
- Why candidates failed: stale (1), apply_conflict (1), stale_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`
- Files changed by rejected candidates: `secp256r1/src/field.rs`
- Median model spend per attempt: 0.4895 USD (over 20 attempts with a recorded cost)

## Target perf:msm_ir

- Attempts: 19 (9 accepted, 8 no candidate, 2 rejected)
- Best accepted effect: 9.17% (ratio 0.9083), episode `d0b327abbfcc`
- Accepted effects: 9.17%, 6.21%, 3.67%, 3.09%, 3.03%, 2.55%, 2.24%, 1.14%, 1.07%
- Why candidates failed: stale_conflict (1), apply_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`, `secp256r1/src/group.rs`
- Files changed by rejected candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.6146 USD (over 19 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I tried one change in secp256r1/src/field.rs, lines 711-712. I routed montgomery_square_wide through reduce_step, the shift-based reduction that multiplication already uses, so squaring would stop using the mac-based reduction.
> Build passed for base and candidate. Outputs were identical. It was slower on every metric. scalar_mul_ir went from 227351891 to 231550141, ratio 1.0185. msm_ir went from 148641174 to 151222076, ratio 1.0174. decode_ir went from 101368473 to 107226830, ratio 1.0578. The required ratio was 0.9900, so it was rejected. I reverted it and submitted nothing.
> In the squaring path the mac-based reduction compiles better than the shifted form. Sharing reduce_step between mul and square does not help here. decode_ir took the largest hit, which fits with decode leaning hardest on squaring.
> Next, look at the other remaining scalar_mul idea, co-Z table normalization in group.rs, with its earlier measurement in hand before spending a session on it.
>
> 2026-10-13, episode `228acd8b35e5` (no candidate)

> Target was msm_ir in secp256r1. First try: in group.rs I folded the per-iteration doubling in multi_scalar_mul_vartime into the first nonzero digit via double_add_mixed, and rewrote double() to get Z3 as (Y+Z)^2 - Y^2 - Z^2. Later tries touched field.rs squaring paths around montgomery_square.
> Evaluation 1 gave msm_ir ratio 1.0152 and scalar_mul_ir 1.0416. Rejected. Evaluation 3, with group.rs and field.rs edits over 28 lines, reached msm_ir 0.9938. That is better than parent but short of the required 0.9900. Evaluation 5 regressed to msm_ir 1.0108 and scalar_mul_ir 1.0185. All three had identical outputs. No candidate was submitted.
> In this field code a squaring costs about the same as a multiply. Swapping a multiply for squarings does not pay and can cost. The gains left in the msm loop structure look smaller than 1%.
> Next, look at the cost of the wnaf table build and add_naf_digit per call rather than at doubling formulas. Also check whether the generator table can be reused across calls.
>
> 2026-10-13, episode `19c2c3313030` (no candidate)

> Tried two changes to speed up decode_ir in secp256r1/src/field.rs. First, removed the Jacobi pre-check from FieldElement::sqrt and verified root^2 == input after the exponent chain instead. Second, swapped in a shift-based Montgomery reduction in the montgomery path around line 711.
> The first change measured decode_ir 101940256 to 133915407, ratio 1.3137. Outputs were identical. scalar_mul_ir and msm_ir were flat at 1.0000. Rejected. The second change measured decode_ir ratio 1.0580, scalar_mul_ir 1.0185, msm_ir 1.0173. It was worse everywhere. Rejected. Both are reverted. No candidate was submitted.
> The Jacobi early exit pays for itself. The bench feeds random compressed X values, and about half are non-residues, so skipping the full chain wins. The existing reduction is already cheaper than the shift-based form on this target.
> Next, look at the exponent chain in sqrt itself, or at the cost inside is_non_residue, rather than removing checks. Read lineage_bench.rs decode inputs again before choosing.
>
> 2026-10-13, episode `ec3021a1f325` (no candidate)

> I tried two changes to cut decode_ir in secp256r1/src/field.rs. First, a hybrid Montgomery reduction step in the squaring path around lines 754-779: shifts for the MODULUS[1] product, a mac for MODULUS[3]. Second, computing n - a directly in the swap branch of the Jacobi loops around lines 333-384.
> The first was rejected. decode_ir went from 100142459 to 104018300, ratio 1.0387. scalar_mul_ir was 1.0151 and msm_ir 1.0139. Outputs were identical. The second was also rejected. decode_ir went from 100142459 to 101272948, ratio 1.0113, with the other metrics unchanged at 1.0000. Neither helped. Both are reverted and nothing was submitted.
> The existing mac form in the reduction already compiles to fewer instructions than the shift version. Hand-shaping the arithmetic loses to LLVM here. The Jacobi swap branch is not where decode spends its time.
> Next, profile decode end to end before editing. Look at the sqrt exponentiation chain and the decompress path in group.rs.
>
> 2026-10-13, episode `c7f3bf02ad09` (no candidate)

> Tried: in secp256r1/src/field.rs I routed montgomery_reduce and montgomery_reduce_words through a new montgomery_reduce_words_wide_shifted. Since -p^-1 = 1 mod 2^64, m = r_i, and the products by the modulus limbs become shifts: m<<32 and m>>32 added at limbs i+1 and i+2, plus the top term at 2^192.
> Results: the first attempt was rejected on decode_ir. Ratio 1.0545, required 0.9900, so decode got slower. scalar_mul_ir 0.9911 and msm_ir 0.9862 were fine in that run. After the third edit (lines 738-740), evaluation 3 on msm_ir was accepted. msm_ir went 152285252 to 147615606, ratio 0.9693. scalar_mul_ir 0.9731, decode_ir 0.9967. Outputs identical, 48 lines changed. Submitted. It is queued and the replays have not run yet.
> Belief: the shift form of the reduction helps the mul-heavy paths. decode is sensitive to how the reduction is wired, and the first version made it worse.
> Next: profile decode separately, starting with the sqrt and decompress paths, and wait for the replay result.
>
> 2026-10-13, episode `6419e9cb3520` (accepted)

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
