# Lessons: anza-cryptography

Lineage `92e06c95a332dd6c8f4e2be5187040ee4f375f60217aaff81ae25fa3c0f8e5ca` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 44 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:scalar_mul_ir

- Attempts: 17 (10 accepted, 1 rejected, 6 no candidate)
- Best accepted effect: 7.99% (ratio 0.9201), episode `8bdb0dda22a9`
- Accepted effects: 7.99%, 4.86%, 3.80%, 3.70%, 3.50%, 3.04%, 2.53%, 2.02%, 1.25%, 1.21%
- Why candidates failed: duplicate (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`, `secp256r1/src/group.rs`
- Files changed by rejected candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.5935 USD (over 17 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 16 (3 rejected, 8 accepted, 5 no candidate)
- Best accepted effect: 17.73% (ratio 0.8227), episode `d87452637ca1`
- Accepted effects: 17.73%, 14.23%, 4.04%, 3.44%, 3.04%, 2.59%, 1.70%, 1.36%
- Why candidates failed: stale (1), apply_conflict (1), stale_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`
- Files changed by rejected candidates: `secp256r1/src/field.rs`
- Median model spend per attempt: 0.4144 USD (over 16 attempts with a recorded cost)

## Target perf:msm_ir

- Attempts: 11 (6 accepted, 4 no candidate, 1 rejected)
- Best accepted effect: 9.17% (ratio 0.9083), episode `d0b327abbfcc`
- Accepted effects: 9.17%, 6.21%, 3.67%, 3.03%, 2.24%, 1.14%
- Why candidates failed: stale_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/group.rs`
- Files changed by rejected candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.5699 USD (over 11 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I changed field subtraction in secp256r1/src/field.rs. First I rewrote sub_limbs as a u128 borrow chain and made the modulus correction in Sub a mask. Second I kept the original and added an early branch in Sub that skips the correction when there is no borrow.
> Evaluation 1 was the u128 chain with the masked correction. It made every metric worse: scalar_mul_ir 1.0367, msm_ir 1.0329, decode_ir 1.0258. Equivalence held, and it was rejected. Evaluation 3 was the early branch. It gave scalar_mul_ir 1.0000, msm_ir 0.9999 and decode_ir 0.9970, against a required 0.9900, so it was also rejected. Neither change helped. I submitted no candidate.
> Subtraction is not where scalar_mul_ir spends its time. The compiler already handles the no-borrow case, and wide u128 arithmetic in sub_limbs costs more than the existing limb code.
> Next I should profile scalar_mul in group.rs to find what dominates. Then I should look at the multiply and reduce path in field.rs rather than subtraction.
>
> 2026-10-13, episode `5950e375d605` (no candidate)

> Tried two changes in secp256r1/src/field.rs aimed at decode_ir. First, in montgomery_reduce_words I replaced mac(r, x, MODULUS[1], x) with the equivalent r + (x << 32). Second, in the u128/u64 tails of invert_canonical I used trailing_zeros-based multi-bit shifts instead of single-bit steps.
> Evaluation 1, the mac change: decode_ir 106405613 to 106354040, ratio 0.9995. scalar_mul_ir and msm_ir both 1.0000. Rejected, required 0.9900. Outputs identical. Evaluation 3, the shift change: decode_ir ratio 1.0066, scalar_mul_ir 1.0059, msm_ir 1.0037. Rejected. Worse on all three. Both edits were reverted. No candidate submitted.
> The compiler already strength-reduces the MODULUS[1] multiply, so that path gives nothing. Variable shifts in invert_canonical cost more than they save, likely from branching. decode cost is probably dominated by sqrt and the pow chain, not these spots.
> Next: profile sqrt and is_non_residue on the decode path. Try a shorter addition chain or fewer square_n calls there.
>
> 2026-10-13, episode `acaf9f6a3c55` (no candidate)

> I tried a cheaper inner loop in the binary Jacobi symbol check that runs before FieldElement::sqrt in secp256r1/src/field.rs. In is_non_residue (4 limbs) and is_non_residue_u192 (3 limbs), the separate compare (ge_limbs or tuple compare) followed by swap and subtract became one subtraction a - n. When it borrows, the operands swap and the new a is the negation of the difference. Two edits, 32 changed lines.
> Sandbox evaluation: decode_ir went from 106521923 to 105053625, ratio 0.9862. scalar_mul_ir and msm_ir were unchanged at ratio 1.0000. Outputs were identical. Guards ok. Submitted, status waiting on independent replays.
> The Jacobi loop is a real but small share of decode_ir. Removing one compare per iteration moved the ratio only to 0.9862. Most decode cost is elsewhere, likely the sqrt exponentiation chain.
> Next: look at the square_n and pow chain inside FieldElement::sqrt and at montgomery_square. Also check whether the Jacobi check can be dropped and the sqrt result verified by squaring instead.
>
> 2026-10-13, episode `c800148042cc` (accepted)

> I changed the doubling in secp256r1/src/group.rs, lines 358-366. It computed Z3 = (Y+Z)^2 - Y^2 - Z^2 instead of 2Y*Z. That trades one field multiply for four additions or subtractions. I read field.rs and the wnaf and scalar mul paths in group.rs before editing.
> Equivalence held and outputs were identical. scalar_mul_ir went from 238772271 to 244988478, ratio 1.0260. msm_ir went from 153362058 to 156065508, ratio 1.0176. decode_ir was unchanged at 1.0000. The required ratio was 0.9900. The change did not help, it made both targets worse. It was rejected and I reverted it. I submitted nothing.
> In this field code, additions and subtractions are not cheap compared to a multiply. Swapping one multiply for four adds loses. Formula tricks that remove a multiply by adding linear ops should be rejected unless the add count stays low.
> Next, look at the field add, sub and reduce_sum paths in field.rs. Making those cheaper may pay off more than counting multiplies. Also check how much of the cost comes from the window and table setup in the scalar mul path.
>
> 2026-10-13, episode `38a6d1d9cb81` (no candidate)

> I replaced the table build in odd_multiples in secp256r1/src/group.rs (lines 701-706, 6 lines became 51) with co-Z additions. P is rescaled once to the Z of 2P. Each addition P_i + 2P also returns 2P rescaled to the Z of the sum. That is 5M+2S per entry instead of a mixed addition plus a Z multiplication, about 9M+3S. Degenerate inputs (2P is the identity, or equal x) take a fallback path.
> The sandbox run on scalar_mul_ir was accepted with identical outputs. scalar_mul_ir went from 236818137 to 233843888, ratio 0.9874. msm_ir went from 160475382 to 156740083, ratio 0.9767. decode_ir stayed at 1.0000. The guard passed with 1 file, 45 lines. This is my own measurement. The replay verdict is still pending.
> Table construction is a small share of scalar_mul. A ratio of 0.9874 is about all the table can give. The main loop of doublings and additions dominates.
> Next I want to look at double and double_add_mixed in group.rs, and at square and square_n in field.rs.
>
> 2026-10-13, episode `5e5e56c840f1` (accepted)

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
