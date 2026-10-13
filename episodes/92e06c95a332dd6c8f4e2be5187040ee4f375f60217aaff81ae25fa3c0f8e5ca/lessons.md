# Lessons: anza-cryptography

Lineage `92e06c95a332dd6c8f4e2be5187040ee4f375f60217aaff81ae25fa3c0f8e5ca` on [https://github.com/anza-xyz/cryptography](https://github.com/anza-xyz/cryptography) at `3efe73c7395f735f073bd25162cdc2fb4333f570`. 14 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:scalar_mul_ir

- Attempts: 7 (5 accepted, 1 rejected, 1 no candidate)
- Best accepted effect: 7.99% (ratio 0.9201), episode `8bdb0dda22a9`
- Accepted effects: 7.99%, 4.86%, 3.80%, 3.70%, 3.50%
- Why candidates failed: duplicate (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`, `secp256r1/src/group.rs`
- Files changed by rejected candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.3862 USD (over 7 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 4 (2 rejected, 2 accepted)
- Best accepted effect: 14.23% (ratio 0.8577), episode `1a50748553f5`
- Accepted effects: 14.23%, 3.04%
- Why candidates failed: stale (1), apply_conflict (1)
- Files changed by accepted candidates: `secp256r1/src/field.rs`
- Files changed by rejected candidates: `secp256r1/src/field.rs`
- Median model spend per attempt: 0.2486 USD (over 4 attempts with a recorded cost)

## Target perf:msm_ir

- Attempts: 3 (3 accepted)
- Best accepted effect: 9.17% (ratio 0.9083), episode `d0b327abbfcc`
- Accepted effects: 9.17%, 3.67%, 2.24%
- Files changed by accepted candidates: `secp256r1/src/group.rs`
- Median model spend per attempt: 0.3134 USD (over 3 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I tried replacing FieldElement::sqrt in secp256r1/src/field.rs. It used a generic bit-by-bit pow over a 32-byte exponent for x^((p+1)/4). For P-256, (p+1)/4 = (2^32 - 1) * 2^222 + 2^190 + 2^94. So I wrote a fixed addition chain. It builds x^(2^32-1) via x^3, x^(2^4-1), x^(2^8-1), x^(2^16-1), then runs square sequences of 32, 96 and 94 with single multiplications. I also cut the old exponent constant block at lines 39-44 down to 1 line.
> Sandbox run on decode_ir went from 176200511 to 151179814, ratio 0.8580. scalar_mul_ir went from 271056098 to 271047304 and msm_ir from 202672055 to 202663246, both ratio 1.0000. Outputs were identical, both builds were ok, and guards were ok. I submitted it with 30 changed lines. Status is waiting on replays.
> Decode cost is dominated by sqrt. The scalar and MSM paths do not touch it.
> Next I want to check invert for the same generic pow pattern and give it a fixed chain too. That would hit scalar_mul_ir and msm_ir.
>
> 2026-10-13, episode `bf7cc3acd68c` (rejected)

> Target was decode_ir in secp256r1/src/field.rs. First I rewrote the reduction path around lines 372-387. In reduce_wide and reduce_sum I then replaced the final conditional subtraction with an addition of a new constant NEG_MODULUS = 2^256 - p, using the existing adc-based add_limbs chain. The input is at least p exactly when that add carries out of 256 bits, or when the fifth limb or incoming carry is set.
> Evaluation 1 was rejected with no improvement. decode_ir went 155858154 to 156218867, ratio 1.0023, so the first rewrite did not help. Evaluation 3 failed to build and the field tests failed with it. Evaluation 5 was accepted with identical outputs. decode_ir went 155858154 to 151122621 (0.9696), scalar_mul_ir 0.9754, msm_ir 0.9753. Submitted, 31 lines, status waiting on replays.
> The sbb borrow chain in the final reduction step costs measurable instructions. The adc chain with a carry-out test is cheaper here.
> Next, check fn sqrt and fn invert in field.rs for the same sbb pattern.
>
> 2026-10-13, episode `292d443465fa` (accepted)

> I replaced the generic pow in FieldElement::sqrt in secp256r1/src/field.rs. It computed x^((p+1)/4) bit by bit over a 32-byte exponent, which meant 256 squarings plus one multiply per set bit. For P-256, (p+1)/4 = (2^32 - 1) * 2^222 + 2^190 + 2^94, so I wrote a fixed addition chain. It builds x^(2^32-1) from x^3, x^(2^4-1), x^(2^8-1) and x^(2^16-1), then does 32 squarings and a multiply by x, and continues from there. I also collapsed lines 31-36, where the old exponent constant lived, to 1 line.
> Sandbox: decode_ir went from 181725992 to 155884481, ratio 0.8578. scalar_mul_ir and msm_ir held at ratio 1.0000. Outputs were identical and guards ok. The candidate is queued for replay, so there is no verdict yet.
> The decode cost here is dominated by the sqrt exponentiation, and scalar mul does not touch it.
> Next I will look at invert, which likely uses the same generic pow, and check whether a fixed chain helps scalar_mul_ir.
>
> 2026-10-13, episode `1a50748553f5` (accepted)

> I worked only on field inversion, invert_canonical in secp256r1/src/field.rs, targeting scalar_mul_ir. I tried three variants. The first shifted by the trailing zero count and divided by 2^k with mac. The second replaced the binary method with a Fermat addition chain. The third computed u - v once instead of a compare followed by a subtract.
> The trailing zero shift measured scalar_mul_ir 286871713 against parent 277968527, ratio 1.0320. That is worse. The Fermat chain measured 296167664, ratio 1.0655, and msm_ir went to 1.0749. That is worse still. The single u - v measured 276041809, ratio 0.9931. A 16 line variant of it measured 276177491, ratio 0.9936. Both missed the 0.9900 threshold. Outputs were identical in every run. I submitted nothing.
> The binary inversion loop is already tight. Branch and limb tweaks inside it buy under 1%. Inversion is probably too small a share of scalar_mul_ir to clear 1% alone.
> Next I would profile group.rs scalar_mul and batch_normalize to see how often invert runs. I would also look at the cost of square and mul.
>
> 2026-10-13, episode `dcd76b62b436` (no candidate)

> I tried three changes in the secp256r1 crate. Two were in field.rs around lines 186-196 and 419-421. Both were reverted. The third was in ProjectivePoint::add_mixed in group.rs. It replaced the doubled-slope variant, r = 2(S2 - Y1) with I = 4H^2, with the standard Jacobian mixed addition using r = S2 - Y1. That saves five field additions per mixed addition.
> Evaluation 1 was rejected. scalar_mul_ir went from 280337095 to 293979244, ratio 1.0487. Evaluation 3 was also rejected, with scalar_mul_ir at ratio 1.0187. The field.rs edits did not help. The add_mixed change gave scalar_mul_ir 0.9921, short of the required 0.9900. On that same change msm_ir went from 214242500 to 209394385, ratio 0.9774, and decode_ir was 0.9997. Outputs were identical. Evaluation 7 targeted msm_ir and was accepted. I submitted it and it is waiting on replays.
> My reading is that field.rs limb arithmetic is already tight, and the cheap wins are in the group formulas.
> Next I will look at double and the wnaf5 table construction in group.rs for redundant additions.
>
> 2026-10-13, episode `a53d95d1d0f5` (accepted)

## Licence

Code excerpts from https://github.com/anza-xyz/cryptography at 3efe73c7395f735f073bd25162cdc2fb4333f570, copyright its authors; the licence was not determined (see the repository).
