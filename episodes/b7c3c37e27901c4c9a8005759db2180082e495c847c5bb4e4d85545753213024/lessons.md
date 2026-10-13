# Lessons: cryptography

Lineage `b7c3c37e27901c4c9a8005759db2180082e495c847c5bb4e4d85545753213024` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 31 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:scalar_arith_ir

- Attempts: 16 (10 accepted, 3 rejected, 3 no candidate)
- Best accepted effect: 44.19% (ratio 0.5581), episode `5c80d802b931`
- Accepted effects: 44.19%, 8.50%, 5.24%, 4.53%, 4.14%, 3.57%, 2.94%, 1.87%, 1.49%, 1.37%
- Why candidates failed: stale_conflict (1), dependency_failed (1), stale (1)
- Files changed by accepted candidates: `secp256r1/src/scalar.rs`
- Files changed by rejected candidates: `secp256r1/src/scalar.rs`
- Median model spend per attempt: 0.3974 USD (over 16 attempts with a recorded cost)

## Target perf:field_arith_ir

- Attempts: 15 (9 accepted, 3 rejected, 3 no candidate)
- Best accepted effect: 4.62% (ratio 0.9538), episode `aa0144b88815`
- Accepted effects: 4.62%, 3.24%, 3.07%, 2.92%, 2.57%, 2.38%, 2.33%, 1.73%, 1.60%
- Why candidates failed: stale (1), stale_conflict (1), dependency_failed (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`
- Files changed by rejected candidates: `secp256r1/src/field.rs`
- Median model spend per attempt: 0.5500 USD (over 15 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I changed montgomery_reduce_words in secp256r1/src/field.rs. The multiply-accumulates against MODULUS[1] became an equivalent shift-add. The edit touched lines 679-694 and 732-733.
> One sandbox run of perf field_arith_ir. field_arith_ir went from 300888536 to 300828336, ratio 0.9998. The gate requires 0.9900, so it was rejected as no_improvement. scalar_arith_ir stayed at 363374994, ratio 1.0000. Outputs were identical and guards were ok. The change did not help, so I reverted both edits and submitted nothing.
> LLVM already strength-reduces the MODULUS[1] multiply. Hand shift-adds in the reduction loop gain nothing. The Montgomery reduction is not where field_arith_ir loses time. The likely hotspot is invert_canonical, the binary-GCD inverse.
> Next, port a batched divsteps_62 inversion with update_fg and update_de, in place of invert_canonical. Budget for it from the start of a session, because it is too large to do safely at the end of one. Check equivalence against the current invert before measuring.
>
> 2026-10-13, episode `4c5c6c889b38` (no candidate)

> I worked on scalar_arith_ir in secp256r1/src/scalar.rs. I made three attempts. The first changed divsteps_62 to use the 6-bit w = -g/f in both branches. The second rewrote k*n[2] in the Montgomery reduction as k*2^64 - k, since n[2] = 2^64-1. The third replaced mul_wide with a product-scanning column version.
> The divsteps change measured 363921926 against parent 363406615, ratio 1.0014, rejected. The reduction change measured 363454216, ratio 1.0001, rejected. The first mul_wide edit failed to build, and the field tests failed with it. The fixed mul_wide built and measured 377658558, ratio 1.0392, which is worse. Outputs were identical in every build that compiled. field_arith_ir stayed at 300933826 every time. None of these changes helped.
> What I believe now: divsteps batch count is not where the cost is. LLVM already emits the k*2^64 - k form on its own. Column-scanning mul_wide costs more than the current version.
> Next time I will profile which function dominates scalar_arith_ir before editing anything, starting with invert and montgomery_square.
>
> 2026-10-13, episode `38caff709c7f` (no candidate)

> Tried: in secp256r1/src/scalar.rs I changed reduce_wide (runs after every Montgomery multiply and square) and reduce_sum. Both decided value >= n with a u128 sbb chain that subtracted n. They now add MODULUS_COMPLEMENT, which is 2^256 - n derived from MODULUS at compile time, with an adc chain. The carry out of 256 bits is set exactly when the low limbs are at least n. Edits at lines 25-26, 376-379 and 552-559. 27 changed lines.
> Measured in my sandbox: scalar_arith_ir went from 379133951 to 363435879, ratio 0.9586. field_arith_ir stayed at 300872297, ratio 1.0000, which is expected since field code was not touched. Outputs were identical and guards passed. Submitted and queued. The verdict waits on independent replays.
> Belief: the final reduction is a real share of scalar cost, and the borrow chain is more expensive than the carry chain here.
> Next: look at the update_de, update_fg and divsteps_62 paths in invert for the same u128 subtract pattern.
>
> 2026-10-13, episode `41714c26837a` (accepted)

> I changed FieldElement::sub in secp256r1/src/field.rs. It no longer computes a full add_limbs and then selects. It now adds p back under a borrow mask with an adc chain. I also tried unrolling add_limbs and sub_limbs into adc/sbb chains, and I tried a 71 to 28 line rewrite around lines 592-662, which I later reverted.
> The masked sub alone measured field_arith_ir 300858615 to 299533446, ratio 0.9956. The gate requires 0.9900, so it was rejected. Outputs were identical and scalar_arith_ir stayed at ratio 1.0000. With the add_limbs/sub_limbs unroll added, the ratio went back to 1.0000 (300846368), so that did not help. The 111-line version measured 0.9959, which is no better. The trimmed 12-line sub repeated 0.9956. I submitted nothing.
> I now think sub is not where field_arith_ir spends its instructions. The limb helpers already compile to roughly the chains I wrote by hand.
> Next I want to profile mul and reduce_wide (lines 320-470) before I touch anything else. That is where a cut of 1% would have to come from.
>
> 2026-10-13, episode `7c4eb425fb90` (no candidate)

> Worked in secp256r1/src/field.rs on the final conditional subtraction. First tries rewrote the add/sub helpers around lines 200-221. Then a change near line 165 that I reverted. Then reduce_wide, at lines 704-710 and 726-727. It now adds 2^256 - p with an adc chain instead of subtracting p with a u128-based sbb chain. Carry out ORed with the fifth limb selects the result.
> Evaluation 1 made field_arith_ir worse, ratio 1.0354, so rejected. Evaluation 3 was 1.0021, rejected. Evaluation 5 was 0.9902 against a required 0.9900, rejected. Evaluation 7 measured 300831181 against parent 308173922, ratio 0.9762, accepted. scalar_arith_ir was flat. Outputs were identical in every run. Submitted with 20 changed lines. It is queued for replay.
> reduce_wide runs after every Montgomery multiply and square, so its cost shows in field_arith_ir. The u128 sbb chain was the expensive part. Rewriting the shared helpers did not help.
> Next: look for other sbb( call sites that could take the same adc trick, and measure each one alone.
>
> 2026-10-13, episode `3fe02f29657f` (accepted)

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
