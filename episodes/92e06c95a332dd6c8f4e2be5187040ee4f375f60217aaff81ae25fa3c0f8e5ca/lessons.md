# Lessons: anza-cryptography

Lineage `92e06c95a332dd6c8f4e2be5187040ee4f375f60217aaff81ae25fa3c0f8e5ca` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 53 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:scalar_mul_ir

- Attempts: 18 (10 accepted, 1 rejected, 7 no candidate)
- Best accepted effect: 7.99% (ratio 0.9201), episode `8bdb0dda22a9`
- Accepted effects: 7.99%, 4.86%, 3.80%, 3.70%, 3.50%, 3.04%, 2.53%, 2.02%, 1.25%, 1.21%
- Why candidates failed: duplicate (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`, `secp256r1/src/group.rs`
- Files changed by rejected candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.6101 USD (over 18 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 18 (3 rejected, 9 accepted, 6 no candidate)
- Best accepted effect: 17.73% (ratio 0.8227), episode `d87452637ca1`
- Accepted effects: 17.73%, 14.23%, 4.04%, 3.71%, 3.44%, 3.04%, 2.59%, 1.70%, 1.36%
- Why candidates failed: stale (1), apply_conflict (1), stale_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`
- Files changed by rejected candidates: `secp256r1/src/field.rs`
- Median model spend per attempt: 0.4264 USD (over 18 attempts with a recorded cost)

## Target perf:msm_ir

- Attempts: 17 (8 accepted, 7 no candidate, 2 rejected)
- Best accepted effect: 9.17% (ratio 0.9083), episode `d0b327abbfcc`
- Accepted effects: 9.17%, 6.21%, 3.67%, 3.03%, 2.55%, 2.24%, 1.14%, 1.07%
- Why candidates failed: stale_conflict (1), apply_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/group.rs`
- Files changed by rejected candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.6040 USD (over 17 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

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

> I tried two changes in secp256r1/src/field.rs aimed at decode_ir. First, in invert_canonical and its 192/128/64-bit tails, I stripped all trailing zeros with one variable shift instead of one bit per step. Second, I dropped the Jacobi pre-check in sqrt.
> The first change measured decode_ir 1.0238, scalar_mul_ir 1.0205 and msm_ir 1.0128. It was rejected against the required 0.9900. The second measured decode_ir 1.3373, with scalar_mul_ir and msm_ir at 1.0000. Outputs were identical in both runs. Neither change helped. Both are reverted, and no candidate was submitted.
> The zero runs stripped in invert_canonical are usually one bit long, so a variable shift costs more than the single-bit step it replaces. The Jacobi pre-check in sqrt pays for itself on the decode path. Removing it made decode much worse.
> Next I would leave the inversion loop structure and the sqrt pre-check as they are. I would look elsewhere in the decode path in group.rs for cost.
>
> 2026-10-13, episode `2e78a5c9173c` (no candidate)

> I tried two changes to secp256r1 scalar multiplication. In group.rs at lines 358-366 I swapped the doubling to the 2001-b form, computing Z3 = (Y+Z)^2 - Y^2 - Z^2 in place of 2Y*Z. In field.rs I rewrote FieldElement::sub so it branches explicitly.
> The doubling change measured worse. scalar_mul_ir went from 233643966 to 238808652, ratio 1.0221, and msm_ir had ratio 1.0129. Outputs were identical. It was rejected. The sub change measured flat. scalar_mul_ir was 233635430 against 233643966, ratio 1.0000, and msm_ir was 0.9999. It was rejected because it missed the 0.9900 bar. Nothing was submitted.
> In this field code a multiplication is not costly enough for the 2001-b swap to pay. The added field additions and subtractions cost more than turning one multiplication into a squaring saves. The branch shape in sub does not move the instruction count.
> Next I will look at the scalar_mul loop structure and the odd_multiples and normalize_co_z paths rather than at single field ops.
>
> 2026-10-13, episode `5b9b5ea7696b` (no candidate)

> Tried speeding up msm_ir in secp256r1. First pass was in field.rs around lines 755-770, then I moved to group.rs. The idea that held: the odd-multiple tables P, 3P, ..., 15P come from a co-Z chain where Z_k = Z_{k-1} * h_k. New odd_multiples_co_z returns the table plus the h_k factors, or None in degenerate cases. New normalize_co_z_chains batch-inverts only the last Z of each table with one shared inversion and recovers earlier inverses by multiplying with h_k.
> Evaluation 1, the field.rs change at 15 lines, did not help: msm_ir ratio 1.0005, rejected. Evaluation 3 reached 0.9947 and evaluation 5 reached 0.9912. Both were rejected against the required 0.9900. Evaluation 7 at 134 lines was accepted with msm_ir 157296445 to 155552933, ratio 0.9889. scalar_mul_ir came in at 0.9993 and decode_ir at 1.0000. Outputs were identical in every run.
> The inversion count in table normalization was the lever. Field-level tweaks were noise. Verdict is pending replay.
> Next, check whether scalar_mul can reuse normalize_co_z_chains.
>
> 2026-10-13, episode `c991056424a2` (accepted)

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
