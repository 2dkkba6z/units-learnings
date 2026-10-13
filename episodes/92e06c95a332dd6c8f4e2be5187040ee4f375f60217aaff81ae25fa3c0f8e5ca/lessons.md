# Lessons: anza-cryptography

Lineage `92e06c95a332dd6c8f4e2be5187040ee4f375f60217aaff81ae25fa3c0f8e5ca` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 28 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:scalar_mul_ir

- Attempts: 10 (7 accepted, 1 rejected, 2 no candidate)
- Best accepted effect: 7.99% (ratio 0.9201), episode `8bdb0dda22a9`
- Accepted effects: 7.99%, 4.86%, 3.80%, 3.70%, 3.50%, 3.04%, 2.53%
- Why candidates failed: duplicate (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`, `secp256r1/src/group.rs`
- Files changed by rejected candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.5693 USD (over 10 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 12 (3 rejected, 7 accepted, 2 no candidate)
- Best accepted effect: 17.73% (ratio 0.8227), episode `d87452637ca1`
- Accepted effects: 17.73%, 14.23%, 4.04%, 3.44%, 3.04%, 2.59%, 1.70%
- Why candidates failed: stale (1), apply_conflict (1), stale_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`
- Files changed by rejected candidates: `secp256r1/src/field.rs`
- Median model spend per attempt: 0.3029 USD (over 12 attempts with a recorded cost)

## Target perf:msm_ir

- Attempts: 6 (4 accepted, 2 no candidate)
- Best accepted effect: 9.17% (ratio 0.9083), episode `d0b327abbfcc`
- Accepted effects: 9.17%, 3.67%, 2.24%, 1.14%
- Files changed by accepted candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.4648 USD (over 6 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I first edited secp256r1/src/group.rs around lines 424-436 and then reverted it. Next I changed field.rs lines 270-278 alone. Evaluation 1 rejected it: scalar_mul_ir went 252121129 to 256031580, ratio 1.0155 against the required 0.9900. That change did not help. I then rewrote invert_canonical in field.rs as Kaliski's almost inverse, x = a^-1 * 2^k mod p with 256 <= k <= 512, in place of binary extended GCD with a modular halving per shifted bit. The invariant p = u*s + v*r keeps r and s at most p, so the loop uses plain shifts and adds, and only the final doubling is reduced. FieldElement::invert folds the 2^k factor back out. Evaluation 3 accepted it. scalar_mul_ir ratio 0.9695, msm_ir 0.9822, decode_ir 0.9652. Outputs were identical, 77 lines changed. It is submitted and queued for replay. I now think inversion cost shows up in all three metrics, decode most. Next I want to profile what remains in scalar_mul besides inversion, and check how batch_normalize calls invert.
>
> 2026-10-13, episode `1d3442810b07` (accepted)

> Target was decode_ir in secp256r1. I read field.rs and group.rs, then tried two changes. First, in the 4-limb and 3-limb Jacobi loops in field.rs I reused the subtraction borrow instead of a separate comparison. In from_compressed in group.rs I skipped the redundant second parity check. Second, I replaced the binary extended-GCD field inversion in field.rs.
> Attempt one gave identical outputs and decode_ir went from 111443694 to 111972236, ratio 1.0047. Rejected. scalar_mul_ir and msm_ir were unchanged at 1.0000. The extra subtraction on the swap path costs more than the comparison it removes. Attempt two also gave identical outputs but was worse: decode_ir ratio 1.0694, scalar_mul_ir 1.0616, msm_ir 1.0353. Rejected. I reverted both and submitted nothing.
> The existing Jacobi loops and the binary GCD inverse are already tight. Removing a comparison does not pay for itself there.
> Next, profile decode to see where the sqrt cost sits before touching the Jacobi or inverse code again.
>
> 2026-10-13, episode `92465cc3a6b2` (no candidate)

> I changed is_non_residue in secp256r1/src/field.rs. It is the binary Jacobi symbol check that runs before FieldElement::sqrt. Once both operands fit in 192 bits, it now hands off to a new 3-limb loop, is_non_residue_u192. That loop hands off to the existing is_non_residue_u128 once both fit in 128 bits. The reduction steps are unchanged: strip trailing zeros, flip the sign for n = 3,5 mod 8 on an odd shift, apply reciprocity. The change is 52 lines in 1 file.
> Sandbox evaluation 1 on decode_ir was accepted with identical outputs. decode_ir went from 116047572 to 113109730, ratio 0.9747. scalar_mul_ir and msm_ir stayed at ratio 1.0000. Guard was ok. This is my own measurement, not a verdict. The candidate is committed and queued for independent replay.
> The Jacobi loop spends time on wide limbs after the operands have already shrunk. Narrowing the width in stages helped decode here.
> Next I will test a different shrink threshold and profile sqrt itself.
>
> 2026-10-13, episode `6ff8f88986d6` (accepted)

> I changed is_non_residue_u128 in secp256r1/src/field.rs. That function is the 128-bit tail of the binary Jacobi symbol check in front of FieldElement::sqrt. Once both operands fit in 64 bits it now hands off to a new u64 loop, is_non_residue_u64. The reduction steps are unchanged: strip trailing zeros, flip the sign for n = 3,5 mod 8 on an odd shift, apply reciprocity on swap, then subtract. The edit replaced 3 lines with 27, and the guard counted 24 changed lines in 1 file.
> In my sandbox, decode_ir went from 116426217 to 114423712, ratio 0.9828. scalar_mul_ir and msm_ir stayed at ratio 1.0000. Outputs were identical and both builds passed. This is my own measurement. The candidate is queued and the replays will decide.
> The 128-bit tail is a real cost in decode. Narrowing the word width pays because the Jacobi loop runs many iterations after the operands shrink.
> Next, look at the wider stage before is_non_residue_u128 for the same kind of handoff. Also check whether sqrt's pow and square_n chain dominates what remains of decode_ir.
>
> 2026-10-13, episode `ea759a26a78f` (accepted)

> I tried two changes in secp256r1/src/field.rs aimed at scalar_mul_ir. The first branches on the borrow in FieldElement::sub instead of always computing the +p correction. The second shifts u and v by their whole trailing-zero count in invert_canonical.
> The sub change measured scalar_mul_ir 252224925 to 252216565, ratio 1.0000. msm_ir was also 1.0000 and decode_ir was 0.9973. It was rejected against the required 0.9900. The compiler already emits a branch there, so the edit changed nothing. The invert_canonical change made things worse. scalar_mul_ir went to 256012652, ratio 1.0150. msm_ir was 1.0089 and decode_ir was 1.0164. Outputs were identical in both runs. I submitted nothing.
> FieldElement::sub is not where the cost is. The single-bit shift loop in invert_canonical beats the variable shift on this workload.
> Next, look at the point formulas in group.rs, such as double and add_mixed, for multiplications or reductions that can be removed. Also check how often invert is actually called per scalar mul before touching it again.
>
> 2026-10-13, episode `21ae6ea34b93` (no candidate)

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
