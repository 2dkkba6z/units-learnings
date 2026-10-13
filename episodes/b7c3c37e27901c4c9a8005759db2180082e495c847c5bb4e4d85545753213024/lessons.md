# Lessons: cryptography

Lineage `b7c3c37e27901c4c9a8005759db2180082e495c847c5bb4e4d85545753213024` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 26 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:scalar_arith_ir

- Attempts: 14 (9 accepted, 3 rejected, 2 no candidate)
- Best accepted effect: 44.19% (ratio 0.5581), episode `5c80d802b931`
- Accepted effects: 44.19%, 8.50%, 5.24%, 4.53%, 3.57%, 2.94%, 1.87%, 1.49%, 1.37%
- Why candidates failed: stale_conflict (1), dependency_failed (1), stale (1)
- Files changed by accepted candidates: `secp256r1/src/scalar.rs`
- Files changed by rejected candidates: `secp256r1/src/scalar.rs`
- Median model spend per attempt: 0.3974 USD (over 14 attempts with a recorded cost)

## Target perf:field_arith_ir

- Attempts: 12 (8 accepted, 3 rejected, 1 no candidate)
- Best accepted effect: 4.62% (ratio 0.9538), episode `aa0144b88815`
- Accepted effects: 4.62%, 3.24%, 3.07%, 2.92%, 2.57%, 2.33%, 1.73%, 1.60%
- Why candidates failed: stale (1), stale_conflict (1), dependency_failed (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`
- Files changed by rejected candidates: `secp256r1/src/field.rs`
- Median model spend per attempt: 0.5615 USD (over 12 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I changed FieldElement addition in secp256r1/src/field.rs. The old code ran an overflowing_add chain, then a separate overflowing_sub chain against p, and combined two carry flags per limb. The new code computes the sum with the existing u128-based adc helper, then adds the constant 2^256 - p with a second adc chain. That chain carries out of 256 bits exactly when the sum is at least p. The edits were three hunks, 39 changed lines in one file.
> Sandbox evaluation on field_arith_ir was accepted. The build was ok on base and candidate, and outputs were identical. field_arith_ir went from 316278104 to 308138536, ratio 0.9743. scalar_arith_ir stayed at 379092957, ratio 1.0000. That is expected, since scalar.rs was not touched. This is my own measurement, not the replay verdict.
> The carry-flag juggling in add was costing instructions. Folding the reduction into one adc chain removes it.
> Next, check whether subtraction and the reduce paths in field.rs can use the same trick. Also check whether scalar.rs addition has the same two-chain pattern.
>
> 2026-10-13, episode `20dfa578104d` (accepted)

> I changed invert_canonical in secp256r1/src/scalar.rs, the Bernstein-Yang divsteps inverse. update_fg now takes a live limb count. After each outer iteration, if the top signed limb of both f and g is pure sign extension (0 or -1), I fold it into the limb below and shrink the length. The matrix application stops touching dead limbs as f and g shrink. The zero test on g and the final sign check on f also use the live length. Two edits, 25 changed lines.
> In my sandbox, scalar_arith_ir went from 384810802 to 379064307, ratio 0.9851. field_arith_ir held at ratio 1.0000 (316360824 to 316359463). Outputs were identical and the guard passed. The candidate is submitted and queued. Replays decide.
> A ratio of 0.9851 says full-width update_fg was a real cost, but a small one. I have not measured where the rest goes. divsteps_62 is my guess.
> Next, apply the same live-length trim to update_de, then profile divsteps_62.
>
> 2026-10-13, episode `11685f5bb3e0` (accepted)

> I tried three changes in secp256r1/src/scalar.rs, all on scalar inversion in invert_canonical (Bernstein-Yang divsteps). The last one makes update_fg take a live length. After each outer iteration, if the top limb of both f and g is 0 or -1, it is folded into the limb below and the length drops. The matrix then stops touching dead limbs.
> Evaluation 1 did not help: scalar_arith_ir 392293207 to 392278807, ratio 1.0000 against a required 0.9900. Evaluation 3 failed to build, and the field tests in lib.rs were listed as failing. Evaluation 5 was accepted in my sandbox: scalar_arith_ir 392293207 to 383410097, ratio 0.9774. field_arith_ir stayed at ratio 1.0000. Outputs were identical. The session still ended with no candidate, because the pending parent it was built on was not accepted.
> I believe the cost of invert_canonical is in update_fg doing full 5-limb work on shrinking values. Tightening only the loop structure in evaluation 1 bought nothing.
> Next: rebase the live-length update_fg onto an accepted parent and resubmit. Then check whether the d/e update can shrink the same way.
>
> 2026-10-13, episode `499107161390` (no candidate)

> I rewrote add_limbs and sub_limbs in secp256r1/src/scalar.rs. Each carry and borrow used to come from two overflowing_add or overflowing_sub results ORed together. Both functions now unroll into four-limb chains that call the u128-based adc and sbb helpers already used by the multiply and reduction code. There were two edits, at lines 348-359 and 358-369, each replacing 12 lines with 7.
> In the sandbox, evaluation 1 was accepted. Both builds succeeded and outputs were identical. The guard passed with 1 file and 30 lines. scalar_arith_ir went from 390202415 to 384835483, ratio 0.9862. field_arith_ir went from 316363154 to 316352989, ratio 1.0000. That is my measurement only. Replays decide.
> These helpers sit under Scalar add, sub, neg, reduce_sum and ge_limbs-style comparisons. The adc/sbb form lowers to flat carry chains. The OR-of-two-flags form did not.
> Next I will look for other overflowing_* carry patterns in scalar.rs and in the field code, then check whether neg can skip a subtraction.
>
> 2026-10-13, episode `f264b398dbd8` (accepted)

> I rewrote divsteps_62 in secp256r1/src/scalar.rs, which sits behind Scalar::invert. It now uses the variable-time divsteps formulation with eta = -delta, following libsecp256k1's modinv64_divsteps_62_var. On each odd step it picks a small multiple w of f, taken from an inverse of f mod 2^6 or 2^4, so that several low bits of g clear at once. The ctz shift that follows then consumes those bits together instead of one per step. The edits were lines 215-255 and 316-320.
> In the sandbox, scalar_arith_ir went from 397652699 to 390226150, ratio 0.9813. field_arith_ir stayed at 316370830, ratio 1.0000. Outputs were identical, both builds passed, and the guard was ok at 80 lines. The candidate is submitted and queued for replay.
> Inversion is a measurable share of scalar_arith_ir, but a 0.9813 ratio means most of the cost is elsewhere. The field arithmetic is untouched.
> Next, profile the rest of scalar_arith_ir in scalar.rs, mul and reduce first.
>
> 2026-10-13, episode `4d8c22780e8d` (accepted)

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
