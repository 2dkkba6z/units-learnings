# Lessons: anza-cryptography

Lineage `92e06c95a332dd6c8f4e2be5187040ee4f375f60217aaff81ae25fa3c0f8e5ca` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 64 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:scalar_mul_ir

- Attempts: 21 (10 accepted, 1 rejected, 10 no candidate)
- Best accepted effect: 7.99% (ratio 0.9201), episode `8bdb0dda22a9`
- Accepted effects: 7.99%, 4.86%, 3.80%, 3.70%, 3.50%, 3.04%, 2.53%, 2.02%, 1.25%, 1.21%
- Why candidates failed: duplicate (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`, `secp256r1/src/group.rs`
- Files changed by rejected candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.6268 USD (over 21 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 22 (3 rejected, 9 accepted, 10 no candidate)
- Best accepted effect: 17.73% (ratio 0.8227), episode `d87452637ca1`
- Accepted effects: 17.73%, 14.23%, 4.04%, 3.71%, 3.44%, 3.04%, 2.59%, 1.70%, 1.36%
- Why candidates failed: stale (1), apply_conflict (1), stale_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`
- Files changed by rejected candidates: `secp256r1/src/field.rs`
- Median model spend per attempt: 0.4895 USD (over 22 attempts with a recorded cost)

## Target perf:msm_ir

- Attempts: 21 (9 accepted, 10 no candidate, 2 rejected)
- Best accepted effect: 9.17% (ratio 0.9083), episode `d0b327abbfcc`
- Accepted effects: 9.17%, 6.21%, 3.67%, 3.09%, 3.03%, 2.55%, 2.24%, 1.14%, 1.07%
- Why candidates failed: stale_conflict (1), apply_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`, `secp256r1/src/group.rs`
- Files changed by rejected candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.6226 USD (over 21 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I edited secp256r1/src/group.rs in five places to cut instruction count in the multi-scalar path. Lines 635-645 and 639-642 sit in the MSM loop. Lines 354-358, 551-554 and 875-878 touch point helpers. I read field.rs for the cost of mul, square and add first.
> Evaluation 1 measured msm_ir at 157955700 to 156729693, ratio 0.9922. It was rejected against the required 0.9900. Evaluation 3 reached 156439596, ratio 0.9904, with scalar_mul_ir at 0.9963. It was rejected. Evaluation 5 reached 156381946, ratio 0.9900, and scalar_mul_ir 0.9961. That was still rejected. All three gave identical outputs and decode_ir stayed at 1.0000. I submitted no candidate.
> Each edit removes well under one percent from msm_ir. Small trims to helpers do not clear the 0.9900 bar on their own. They add up only slowly.
> Next I will change the MSM structure itself, for example the window width or table reuse across scalars, rather than shaving single field operations.
>
> 2026-10-14, episode `0167cec0e247` (no candidate)

> Target was decode_ir in secp256r1. I read the sqrt path in secp256r1/src/field.rs and the decode bench in lineage/examples/lineage_bench.rs. The sqrt addition chain is 253 squarings and 7 multiplications, and squaring was tuned in earlier generations, so I left it alone. The bench sends 3000 random compressed x values, and about half of them are non-residues, so the Jacobi pre-check stays. I edited the 4-limb and 3-limb binary Jacobi loops in field.rs. The subtract-then-negate step became a top-limb comparison.
> Sandbox result: rejected. decode_ir went from 101947078 to 102424960, ratio 1.0047, against a required 0.9900. scalar_mul_ir and msm_ir were unchanged at ratio 1.0000. Outputs were identical and the guard passed. The change made decode slightly slower. I reverted the edits and submitted nothing.
> My current view is that decode cost sits in sqrt and Jacobi, and the branch shape inside the Jacobi loop is not the bottleneck.
> Next I want to measure the per-iteration cost of the Jacobi loop directly. I also want to check whether the full-width 4-limb phase can hand off to the 3-limb phase earlier.
>
> 2026-10-13, episode `0f924faaeabc` (no candidate)

> Tried two things on secp256r1 scalar mul. In group.rs I changed the single-scalar table normalization so the co-Z chain is inverted through its factors and the base inverse is recovered as 2Y/z2. Then in field.rs I added multi-bit trailing-zero stripping to the u128/u64 inversion tails.
> Evaluation 1, the normalization alone: scalar_mul_ir went from 227377278 to 226009005, ratio 0.9940, with identical outputs. msm_ir and decode_ir held at 1.0000. Rejected, because the floor is 0.9900. Evaluation 3, normalization plus the tail stripping: scalar_mul_ir 1.0007, msm_ir 1.0044, decode_ir 1.0075. Every metric got worse. I reverted the field.rs change. Nothing was submitted.
> The co-Z inversion trick is a real gain, but it is too small to count on its own. Trailing-zero stripping in the inversion tails costs more in branches than it saves in shifts.
> Next, keep the group.rs normalization and pair it with a second saving in the scalar_mul path, so that together they clear 0.9900. Leave the inversion tails alone.
>
> 2026-10-13, episode `f96778b1a3b8` (no candidate)

> I edited multi_scalar_mul_vartime_unchecked in secp256r1/src/group.rs. Each iteration now folds its doubling into the addition of the first nonzero digit through double_add_mixed. Then in secp256r1/src/field.rs I made Sub branch on the borrow instead of selecting.
> Evaluation 1 was the group.rs change alone. msm_ir went from 147884722 to 146940479, ratio 0.9936. scalar_mul_ir and decode_ir stayed at 1.0000 and outputs were identical. It was rejected because the required ratio is 0.9900. Evaluation 3 added the field Sub branch. msm_ir measured 146952467, ratio 0.9937, and decode_ir was 1.0001. The Sub change did not help, and that run was rejected the same way. I gave up without submitting.
> The doubling fold gives a real gain on msm_ir, but it is small. Field Sub is not where msm spends its time. Single-step tweaks to the MSM loop will not reach 0.99 by themselves.
> Next I will profile msm_ir and split its cost between window table construction and the doubling chain. Then I will try a structural change, such as a different window width or a Pippenger path, with the double_add_mixed fold stacked on top.
>
> 2026-10-13, episode `def4be436efb` (no candidate)

> I targeted decode_ir in the secp256r1 crate. I read secp256r1/src/field.rs around the sqrt and pow code and secp256r1/src/group.rs lines 110-165. Then I replaced lines 139-143 of group.rs, 5 lines with 10, on the point decode path.
> The sandbox evaluation rejected it as no_improvement. decode_ir went from 100132414 to 100019328, ratio 0.9989, and the required ratio was 0.9900. scalar_mul_ir stayed at 227414957 and msm_ir at 150600370, both ratio 1.0000. Base and candidate both built, the outputs were identical, and the guard passed. The change did not help in any way that counts. I did not submit a candidate.
> Decode cost is not in the code around group.rs lines 139-143. The field sqrt in field.rs is the likely bulk, since it is an exponentiation. Trimming the logic around that call barely moves the count.
> Next I want to work on the sqrt exponentiation chain in field.rs directly, using a shorter addition chain or fewer squarings. After that, check whether decode calls the inversion or the residue check more than once.
>
> 2026-10-13, episode `f09d0d2306b0` (no candidate)

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
