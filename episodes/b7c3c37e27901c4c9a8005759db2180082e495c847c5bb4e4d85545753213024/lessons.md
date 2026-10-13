# Lessons: cryptography

Lineage `b7c3c37e27901c4c9a8005759db2180082e495c847c5bb4e4d85545753213024` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 40 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:scalar_arith_ir

- Attempts: 21 (11 accepted, 3 rejected, 7 no candidate)
- Best accepted effect: 44.19% (ratio 0.5581), episode `5c80d802b931`
- Accepted effects: 44.19%, 8.50%, 5.24%, 4.53%, 4.14%, 3.57%, 2.94%, 2.56%, 1.87%, 1.49%, 1.37%
- Why candidates failed: stale_conflict (1), dependency_failed (1), stale (1)
- Files changed by accepted candidates: `secp256r1/src/scalar.rs`
- Files changed by rejected candidates: `secp256r1/src/scalar.rs`
- Median model spend per attempt: 0.4864 USD (over 21 attempts with a recorded cost)

## Target perf:field_arith_ir

- Attempts: 19 (10 accepted, 3 rejected, 6 no candidate)
- Best accepted effect: 4.62% (ratio 0.9538), episode `aa0144b88815`
- Accepted effects: 4.62%, 3.24%, 3.07%, 2.92%, 2.57%, 2.38%, 2.33%, 1.73%, 1.60%, 1.01%
- Why candidates failed: stale (1), stale_conflict (1), dependency_failed (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`
- Files changed by rejected candidates: `secp256r1/src/field.rs`
- Median model spend per attempt: 0.5541 USD (over 19 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I rewrote mul_wide in secp256r1/src/scalar.rs to sum partial products column by column instead of row by row. mul_wide sits under every scalar multiply. I tried two versions, one with a loop and one unrolled.
> Both built and both gave identical outputs. Neither was faster. The loop version took scalar_arith_ir from 353968194 to 591693167 (ratio 1.6716). The unrolled version took it from 353968194 to 365825823 (ratio 1.0335). The required ratio was 0.9900. field_arith_ir stayed within limits both times, at 1.0055 and 1.0000. I gave up and reverted scalar.rs to its original state. Nothing was submitted.
> The row-by-row schoolbook in mul_wide is already cheap in instruction count. Column-wise accumulation adds carry handling that costs more than it saves, and a loop makes it much worse.
> Next, I would look at the reduction step after mul_wide, or at the Montgomery conversions found by the montgomery_mul and R2 search, rather than the product itself. I would read the IR count per call before editing.
>
> 2026-10-13, episode `188e29342870` (no candidate)

> I rewrote montgomery_mul in secp256r1/src/scalar.rs in CIOS form, interleaving the multiplication with the reduction instead of doing mul_wide followed by montgomery_reduce. The edit touched lines 407-435 and grew the function by a few lines.
> One sandbox evaluation. Builds were ok and outputs identical. scalar_arith_ir went from 354085962 to 357804802, ratio 1.0105, against a required 0.9900. field_arith_ir was unchanged at 297815232. The change did not help. It made scalar arithmetic slower. I reverted it, so the repository is back at the parent. No candidate was submitted.
> The existing separate wide-multiply then reduce path is already competitive at the instruction level. Interleaving added carry handling that cost more than it saved. I could not find another change in scalar.rs that I expected to clear the 1% minimum.
> Next, look at how often invert and square call montgomery_mul and whether a dedicated square path cuts instructions. Profile per-function counts before editing.
>
> 2026-10-13, episode `65e64d3445ab` (no candidate)

> Worked on secp256r1/src/scalar.rs to cut instruction count on scalar_arith_ir. In montgomery_reduce_words the low word of r_i + k*n0 is zero by construction of k, so its carry out is hi(k*n0) + (r_i != 0). I replaced the full mac on n0 in each of the four rounds with a high-product helper, low_word_carry. I also added early returns in reduce_wide and reduce_sum for the no-top-carry case.
> Evaluation 1 was rejected: ratio 0.9940 against a required 0.9900. Evaluation 3 was rejected at 0.9906. Evaluation 5 was accepted: scalar_arith_ir went from 363385810 to 353957627, ratio 0.9741. field_arith_ir was 0.9999. All three had identical outputs. Submitted at 34 changed lines; replay verdict pending.
> The reduction loop carries real redundant work. The partial edits on their own were not enough; the combination is what crossed the line.
> Next: look at the square path and invert for the same zero-low-word carry pattern. field.rs may have the same reduce structure.
>
> 2026-10-13, episode `fba4b8230200` (accepted)

> I worked on secp256r1/src/scalar.rs to cut scalar_arith_ir. The first attempt changed the four Montgomery reduction rounds to use a carry shortcut: the carry out of r_i + lo(k*n0) is r_i != 0. It also made from_bytes_reduced a single chain. The second attempt replaced the n2/n3 macs with the identity n2 + n3*2^64 = 2^128 - 2^96 + 2^64 - 1, using a borrow chain.
> Evaluation 1 had scalar_arith_ir going from 363402947 to 360973190, ratio 0.9933. The bar is 0.9900, so it was rejected. field_arith_ir held at 1.0000 and outputs were identical. The from_bytes_reduced part added nothing. Evaluation 3 was worse: scalar_arith_ir went to 390976327, ratio 1.0759, and field_arith_ir was 1.0054. I did not submit a candidate.
> The carry shortcut is real but small. The borrow-chain rewrite costs more than the macs it replaces.
> Next time, keep the carry shortcut and look for a second saving in the reduction loop or in invert to get past 0.9900.
>
> 2026-10-13, episode `e984ad4dea1f` (no candidate)

> Tried three changes in secp256r1/src/scalar.rs, in montgomery_reduce_words and the square path. First, the carry out of r_i + k*n0 computed as hi(k*n0) + (r_i != 0). Second, that plus a mac-chain square. Third, the multiply by n2 = 2^64-1 replaced with a shift and subtract.
> The carry trick took scalar_arith_ir from 363415006 to 360985014, ratio 0.9933, outputs identical. Rejected, the bar is 0.9900. Adding the mac-chain square gave 361240954, ratio 0.9940, worse than the carry trick alone. The n2 shift and subtract changed the count by 0 instructions. field_arith_ir stayed at 297873599 in every run. No candidate submitted.
> LLVM already lowers the all-ones multiply well. Carry handling in the reduction loop alone does not close the gap to 0.9900.
> Next: profile scalar_arith_ir by function before editing. Check invert and the mul path outside montgomery_reduce_words. Read the bench mix in lineage/examples/lineage_bench.rs to see which ops dominate.
>
> 2026-10-13, episode `ca6d6aa8dbad` (no candidate)

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
