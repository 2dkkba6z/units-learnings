# Lessons: anza-cryptography

Lineage `92e06c95a332dd6c8f4e2be5187040ee4f375f60217aaff81ae25fa3c0f8e5ca` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 38 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:scalar_mul_ir

- Attempts: 13 (8 accepted, 1 rejected, 4 no candidate)
- Best accepted effect: 7.99% (ratio 0.9201), episode `8bdb0dda22a9`
- Accepted effects: 7.99%, 4.86%, 3.80%, 3.70%, 3.50%, 3.04%, 2.53%, 1.21%
- Why candidates failed: duplicate (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`, `secp256r1/src/group.rs`
- Files changed by rejected candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.6268 USD (over 13 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 14 (3 rejected, 7 accepted, 4 no candidate)
- Best accepted effect: 17.73% (ratio 0.8227), episode `d87452637ca1`
- Accepted effects: 17.73%, 14.23%, 4.04%, 3.44%, 3.04%, 2.59%, 1.70%
- Why candidates failed: stale (1), apply_conflict (1), stale_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`
- Files changed by rejected candidates: `secp256r1/src/field.rs`
- Median model spend per attempt: 0.3634 USD (over 14 attempts with a recorded cost)

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

> I tried three edits in secp256r1/src/field.rs. First, a branching FieldElement::sub that adds the modulus only when the subtraction borrows. Second, removing the Jacobi pre-check in sqrt and checking root^2 == self instead. Third, a later variant of the field.rs changes, 26 guarded lines in total.
> The branching sub gave msm_ir 0.9999, decode_ir 0.9971, scalar_mul_ir 1.0000. It did not help, and the compiler already avoids most of that work. Dropping the Jacobi pre-check made decode_ir 1.3304, much worse. The third attempt gave decode_ir 0.9986, msm_ir 0.9993, scalar_mul_ir 1.0013. All three had identical outputs. All three were rejected against the required 0.9900. I submitted nothing.
> On this tip, sub is already close to what the compiler can do. The Jacobi pre-check in sqrt pays for itself on the decode inputs, so keep it.
> Next I will look at mul and montgomery_reduce in field.rs, and at the decode path itself, rather than sub or sqrt.
>
> 2026-10-13, episode `33575a4538d3` (no candidate)

> Tried two decode_ir changes in secp256r1. First, in the Montgomery reduction in field.rs, I replaced the MODULUS[1] multiply-accumulate with a shift-by-32 add. In from_compressed in group.rs I now read the parity of y once through a new FieldElement::is_odd, so there is no second to_be_bytes. Second, a larger edit across field.rs, 127 guarded lines in total.
> Evaluation 1 rejected it. decode_ir went 105638585 to 105548186, ratio 0.9991 against a required 0.9900. scalar_mul_ir and msm_ir stayed at 1.0000. Evaluation 3 was worse. decode_ir ratio was 1.0189, scalar_mul_ir 1.0172, msm_ir 1.0103. Outputs were identical both times. Neither change helped enough, and the second one regressed. Nothing was submitted.
> On this tip the reduction tweak and the duplicate byte conversion are not where decode time goes. Adding code to the field arithmetic costs more than it saves.
> Next, profile decode itself before editing again. Look at the sqrt and pow chain in field.rs and at the on-curve check in group.rs.
>
> 2026-10-13, episode `976032ef6ce6` (no candidate)

> I tried two things on secp256r1. In group.rs I kept the width-5 odd-multiple table in Jacobian form and used full additions in place of batch normalization plus mixed additions. In field.rs I replaced the MODULUS[1] multiply-accumulate in the Montgomery reduction with a shift-by-32 add, then pointed later edits at decode_ir.
> The Jacobian table measured scalar_mul_ir 250998873 vs 241702224, ratio 1.0385. Worse. The reduction change gave scalar_mul_ir 1.0016, msm_ir 1.0004, decode_ir 0.9935. Two more decode_ir attempts both reached 0.9911 against the required 0.9900. Outputs were identical every time. Nothing was submitted.
> The table inversion is cheaper than the extra 4M+1S per full addition, so batch normalization stays. The reduction is already tight. The shift trick moves decode_ir slightly and does not help scalar_mul_ir.
> Next, look at decode_ir through the sqrt and pow chain in field.rs. The square_n path there is where the remaining distance to 0.9900 most likely sits.
>
> 2026-10-13, episode `9fade3da8f06` (no candidate)

> I tried widening the generator signed NAF in double_scalar_mul_vartime_unchecked in secp256r1/src/group.rs from width 8 to width 10. That needed a 256-entry cached affine table and i16 digits instead of i8. Seven small edits, 22 lines in one file.
> Sandbox result: outputs identical. msm_ir went from 165224170 to 164228634, ratio 0.9940. The bar was 0.9900, so it was rejected as no improvement. scalar_mul_ir stayed at ratio 1.0000 and decode_ir also stayed at 1.0000. The change did not help enough. I did not submit it.
> What I now believe: the gain from wider generator digits is mostly eaten by building the larger table. My rough estimate is that building the 256 entries costs about as much as the ~5 generator additions it saves. I have not measured that split.
> Next: check whether the generator table is rebuilt per call. If it is, make it a precomputed constant so the wider window costs nothing at runtime. Otherwise look at the variable-point side, odd_multiples and batch normalization, for the msm_ir cost.
>
> 2026-10-13, episode `59390ca6647e` (no candidate)

> I replaced the fixed 4-bit windows in double_scalar_mul_vartime_unchecked in secp256r1/src/group.rs with signed NAF digits for both scalars. The old code added a table entry about every 4 bits per scalar and built its point table with 15 additions. The generator scalar now uses width 8 against a cached affine table of G, 3G, ..., 127G. That table is built once in a OnceLock with a single batch normalization. The edits touched 86 lines in one file and removed the old table code near line 845.
> In the sandbox, msm_ir went from 182453624 to 164677292, ratio 0.9026. scalar_mul_ir moved from 241616496 to 241674003, ratio 1.0002. decode_ir held at 1.0000. Outputs were identical and the guards passed.
> The double-scalar path was spending its time on additions, not doublings. Fewer nonzero digits plus a precomputed affine table is where the gain came from.
> Next I want to check whether the variable scalar side benefits from a wider NAF with batch normalized odd multiples. scalar_mul_ir still uses the old windowing and could take the same treatment. The verdict waits on replay.
>
> 2026-10-13, episode `1b1a4a65a871` (rejected)

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
