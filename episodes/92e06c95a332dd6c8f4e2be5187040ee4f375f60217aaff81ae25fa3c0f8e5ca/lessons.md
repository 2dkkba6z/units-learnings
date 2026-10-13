# Lessons: anza-cryptography

Lineage `92e06c95a332dd6c8f4e2be5187040ee4f375f60217aaff81ae25fa3c0f8e5ca` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 56 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:scalar_mul_ir

- Attempts: 19 (10 accepted, 1 rejected, 8 no candidate)
- Best accepted effect: 7.99% (ratio 0.9201), episode `8bdb0dda22a9`
- Accepted effects: 7.99%, 4.86%, 3.80%, 3.70%, 3.50%, 3.04%, 2.53%, 2.02%, 1.25%, 1.21%
- Why candidates failed: duplicate (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`, `secp256r1/src/group.rs`
- Files changed by rejected candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.6268 USD (over 19 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 19 (3 rejected, 9 accepted, 7 no candidate)
- Best accepted effect: 17.73% (ratio 0.8227), episode `d87452637ca1`
- Accepted effects: 17.73%, 14.23%, 4.04%, 3.71%, 3.44%, 3.04%, 2.59%, 1.70%, 1.36%
- Why candidates failed: stale (1), apply_conflict (1), stale_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`
- Files changed by rejected candidates: `secp256r1/src/field.rs`
- Median model spend per attempt: 0.4270 USD (over 19 attempts with a recorded cost)

## Target perf:msm_ir

- Attempts: 18 (9 accepted, 7 no candidate, 2 rejected)
- Best accepted effect: 9.17% (ratio 0.9083), episode `d0b327abbfcc`
- Accepted effects: 9.17%, 6.21%, 3.67%, 3.09%, 3.03%, 2.55%, 2.24%, 1.14%, 1.07%
- Why candidates failed: stale_conflict (1), apply_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`, `secp256r1/src/group.rs`
- Files changed by rejected candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.6093 USD (over 18 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

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

> I tried two edits in secp256r1/src/group.rs. The first changed table normalization in mul_scalar_vartime_unchecked to a co-Z form. It does one inversion of the last Z and recovers the earlier inverses from the chain factors. The second changed double() so it checks only whether Z3 is zero, replacing two separate checks.
> Evaluation 1 measured the co-Z normalization. scalar_mul_ir went from 233620507 to 232191589, ratio 0.9939, with identical outputs. That is short of the 0.9900 bar, so it was rejected. msm_ir and decode_ir held at 1.0000. Evaluation 3 added the double() change and was worse. scalar_mul_ir came in at ratio 1.0003 and msm_ir at 1.0037. The double() edit did not help. I submitted nothing.
> Table normalization costs too little of scalar_mul_ir for the inversion count to clear 1 percent. The branch checks in double() are not the bottleneck either.
> Next I want to profile the main loop of mul_scalar_vartime_unchecked, the wnaf width and the add path, before touching normalization again.
>
> 2026-10-13, episode `463665d7b14c` (no candidate)

> I tried to cut msm_ir in secp256r1/src/group.rs. Around lines 635-645 I folded the doubling in the msm loop into the addition of the first nonzero digit, using double_add_mixed. I read the field mul and square code in field.rs, and the wNAF and mixed-addition paths, looking for other targets.
> The sandbox evaluation gave identical outputs. msm_ir went from 157248331 to 156306458, ratio 0.9940. The bar is 0.9900, so it was rejected as no improvement. scalar_mul_ir and decode_ir held at ratio 1.0000. Guard ok. The change did not help enough, so I submitted nothing.
> On this path I found little left to take. Table normalization, wNAF recoding, window width, field mul/square and the mixed-addition formulas already look near optimal. The doubling fold alone reaches 0.9940 and no further.
> Next, combine the double_add_mixed fold with a second saving. Or look outside the msm inner loop, at setup or table build cost, to close the remaining gap to 0.9900.
>
> 2026-10-13, episode `73abe57dd895` (no candidate)

> I tried to cut the cost of normalizing the odd-multiple table in scalar_mul in secp256r1/src/group.rs. The table comes out of a co-Z chain. So I inverted only the last Z and recovered the earlier inverses from the chain factors, replacing per-point normalization. I also tried precomputing a negated table.
> Evaluation 1 with the co-Z normalization gave scalar_mul_ir 233573339 to 232131273, ratio 0.9938. The required ratio is 0.9900, so it was rejected. Outputs were identical. msm_ir and decode_ir stayed at 1.0000. Evaluation 3 added the negated table and got 232196719, ratio 0.9941. That is slightly worse, so I reverted it. No candidate was submitted.
> The table normalization is real work but a small share of scalar_mul. Removing inversions there moves the total by well under one percent. The negated table does not pay for itself.
> Next, look past table setup at the main loop: the wnaf5 digit handling and the per-step add and double costs, plus montgomery_square and montgomery_mul in field.rs.
>
> 2026-10-13, episode `e80009b8a7f3` (no candidate)

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
