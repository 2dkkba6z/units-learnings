# Lessons: cryptography

Lineage `b7c3c37e27901c4c9a8005759db2180082e495c847c5bb4e4d85545753213024` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 4 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:scalar_arith_ir

- Attempts: 3 (2 accepted, 1 rejected)
- Best accepted effect: 5.24% (ratio 0.9476), episode `793797762f60`
- Accepted effects: 5.24%, 2.94%
- Why candidates failed: stale_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/scalar.rs`
- Files changed by rejected candidates: `secp256r1/src/scalar.rs`
- Median model spend per attempt: 0.4000 USD (over 3 attempts with a recorded cost)

## Target perf:field_arith_ir

- Attempts: 1 (1 accepted)
- Best accepted effect: 3.07% (ratio 0.9693), episode `f44efdebcf29`
- Files changed by accepted candidates: `secp256r1/src/field.rs`
- Median model spend per attempt: 0.6173 USD (over 1 attempt with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I replaced FieldElement::invert in secp256r1/src/field.rs. The old code was a binary extended Euclidean inversion. It halved a cofactor mod p for every removed bit, compared u and v with a full subtraction, then subtracted again, and finished with a Montgomery multiply by R2. The new code is the almost Montgomery inverse (Kaliski). Cofactors are doubled instead of halved, so they need plain multi-bit shifts.
> Evaluation 1 was rejected. field_arith_ir went from 385718017 to 456079202, ratio 1.1824. That first version was slower. Evaluation 3 was also rejected, at 389947867, ratio 1.0110. Evaluation 5 was accepted in sandbox at 373839991, ratio 0.9692. scalar_arith_ir stayed at ratio 1.0000. Outputs were identical in all three. The submission is 104 changed lines, queued for replay.
> invert carries a real share of field_arith_ir. The shape of the inner loop decides it, not the algorithm name.
> Next: profile the remaining correction loop and the final R2 multiply, and check whether sqrt or pow can share the gain.
>
> 2026-10-13, episode `f44efdebcf29` (accepted)

> I replaced Scalar::invert in secp256r1/src/scalar.rs. The old version raised the input to n-2 with a 5-bit sliding window, about 256 Montgomery squarings plus multiplies. The new one follows FieldElement::invert in field.rs. It converts out of Montgomery form, runs a binary extended Euclidean inversion modulo n in invert_canonical, then multiplies by R2. I removed the window constants and table code.
> Sandbox run on scalar_arith_ir: parent 919880890, candidate 468041036, ratio 0.5088. field_arith_ir went from 386535490 to 385713132, ratio 0.9979. Outputs were identical. Guards were ok. The change was 120 lines in 1 file. Submitted and queued for replay.
> The scalar bench is dominated by inversion. The exponentiation path was the cost, not the Montgomery multiply.
> Next, check whether the helpers now duplicated across scalar.rs and field.rs can be shared without hurting either metric. Also look at what remains in scalar_arith_ir after inversion.
>
> 2026-10-13, episode `921407f9d2d2` (rejected)

> I tried taking the runtime bit scan out of Scalar::invert in secp256r1/src/scalar.rs. The old code walked the lower 128 bits of the constant exponent n - 2 at runtime and worked out sliding window widths and values. A const fn now builds the same window decomposition at compile time as a list of (squarings, table index) steps. The runtime loop only executes those steps. The squaring and multiplication sequence is unchanged. The edits were lines 125-152 (28 lines down to 5) and lines 212-215 (4 lines up to 49).
> Sandbox evaluation on scalar_arith_ir: parent 871628637, candidate 846038191, ratio 0.9706. field_arith_ir went from 385681978 to 386505234, ratio 1.0021, which is within guard limits. Outputs were identical. Submitted, status waiting on replays.
> The bit testing and window bookkeeping were a measurable part of invert. The arithmetic itself did not change.
> Next I will check whether the upper half of the exponent loop can use the same precomputed step list, and look at the window table size.
>
> 2026-10-13, episode `5ed23cc9b7a2` (accepted)

> Tried: Scalar::invert in secp256r1/src/scalar.rs raised the input to n-2 with a generic 5-bit sliding window over all 256 bits. The upper 128 bits of n-2 are ffffffff 00000000 ffffffff ffffffff. I replaced that half with a fixed chain. x^(2^32-1) is built from the existing odd-power table (x^31 and x^3), then shifted in with 64 and 32 squarings. The rationale estimate was about 7 multiplications instead of about 20. Two edits, 23 changed lines.
> Measured in my sandbox, not a verdict: scalar_arith_ir went from 919878695 to 871634620, ratio 0.9476. field_arith_ir went from 386499481 to 385675822, ratio 0.9979. Outputs identical, guards ok, both builds ok. Submitted, status queued for replay.
> Belief: the lower 128 bits of n-2 have no structure, so the window still has to cover them. The fixed upper half is where the cheap gain was.
> Next: check whether the lower-half window size is still optimal for 128 bits instead of 256. Look for other invert or pow paths with constant exponents.
>
> 2026-10-13, episode `793797762f60` (accepted)

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
